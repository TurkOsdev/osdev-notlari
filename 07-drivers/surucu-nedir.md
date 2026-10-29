# Sürücü nedir, kernel içinde nerede durur

Sürücü (driver), kernel'in bir aygıtla konuşmak için kullandığı kod katmanıdır. Üst katmanlardan gelen "bu dosyadan 512 bayt oku", "bu paketi gönder" veya "ekrana bu pikseli çiz" isteklerini aygıtın anladığı register yazmalarına, kuyruk girdilerine ve interrupt'lara çevirir. Sürücü olmasaydı her dosya sistemi, her ağ protokolü ve her ekran çözünürlüğü aynı diske, ağ kartına ve ekran kartına ayrı ayrı erişmek zorunda kalırdı.

Sürücünün asıl işi yalnızca "donanıma yazmak" değildir. Aygıtın durumunu takip etmek, işi sıraya koymak, hataları ve zaman aşımlarını yönetmek, interrupt ile gelen veriyi güvenli biçimde üst katmana aktarmak ve aygıt kaybolduğunda veya uyku moduna girdiğinde tutarlı bir durum bırakmak da sürücünün sorumluluğudur. Bu yüzden sürücü, kernel'in en çok hata yapılan ve en çok donanım ayrıntısı taşıyan parçalarından biridir.

## Ön koşullar

- [Kernel nedir](../04-kernel/kernel-nedir.md)
- [MMIO ve port I/O](../01-mimari/mmio-ve-port-io.md)
- [IDT (Interrupt Descriptor Table)](../04-kernel/idt.md)

## Kapsam

Bu not x86-64 ve PC platformunu esas alır. Sürücü modelinin kavramları mimariden bağımsızdır; ARM64 ve RISC-V farkları ayrıca belirtilir. Kod örnekleri C ile yazılmıştır ve gerçek bir kernel API'si değil, yapıyı göstermek için kısaltılmıştır.

## Problem

Bir bilgisayarda aynı işi yapan birden çok aygıt olabilir: farklı üreticilerin ağ kartları, farklı denetleyicilere bağlı diskler, farklı klavyeler. Üst katmanların her aygıtın register düzenini bilmesi sürdürülemez. Yeni bir aygıt eklendiğinde dosya sistemini, ağ yığınını ve scheduler'ı değiştirmek gerekirdi.

Bu çoğalmayı önlemenin yolu, aygıta özgü bilgiyi tek bir yerde toplamak ve üst katmana sabit bir arayüz sunmaktır. Sürücü bu sınırı çizer:

- Üst katman "blok oku" der, diskin hangi komutla okunacağını bilmez.
- Sürücü "blok oku" isteğini aygıtın komut register'ına, DMA adresine ve tamamlanma interrupt'ına çevirir.
- Aygıt değiştiğinde yalnızca sürücü değişir.

Ancak sınırın yeri önemlidir. Sürücü, üst katmanın politikasını (hangi dosyanın okunacağı, hangi kullanıcının izni olduğu) bilmemelidir. Aygıtın mekanizmasını bilmeli, kararı başkasına bırakmalıdır. Bu ayrım bozulduğunda sürücü kısa sürede dosya sistemi veya güvenlik kontrolü de yapan bir kod yığınına dönüşür.

## Donanım seviyesinde

Bir aygıt, işlemciye genellikle dört arayüz üzerinden bağlanır:

| Arayüz | Ne taşır | Tipik kullanım |
| --- | --- | --- |
| Register'lar | Komut, durum, yapılandırma | Aygıtı başlatma, işi başlatma, durum okuma |
| Veri penceresi | Veri tamponu veya FIFO | Küçük veri transferi, PIO |
| Interrupt | Aygıtın dikkat istemesi | İş tamamlandı, veri geldi, hata oluştu |
| DMA | Bellek ile aygıt arasında doğrudan transfer | Disk, ağ, ses, görüntü |

Register'lar çoğu zaman üç gruba ayrılır: kontrol register'ları (aygıta ne yapacağını söyler), durum register'ları (aygıtın ne yaptığını bildirir) ve veri register'ları (transfer edilen baytları taşır). Bazı aygıtlarda interrupt durum ve maske register'ları ayrıdır; hangi olayın interrupt ürettiğini görmek için önce durum register'ı okunur, sonra istenmeyen olay maskelenir.

Aygıtla konuşma genellikle bir el sıkışma protokolüdür. Sürücü bir komut yazar, aygıtın meşgul bitini bekler, sonucu okur ve hatayı kontrol eder. Bu bekleme sonsuz döngüye bırakılmamalıdır; aygıt yoksa veya arızalandıysa okunan değer `0xFF` gibi anlamsız bir sabit olabilir. Her donanım erişiminde zaman aşımı bulunmalıdır.

Aygıtlar keşif biçimine göre üçe ayrılır:

- **Numaralandırılabilir veri yolları.** PCIe ve USB, takılı aygıtları kendileri bildirir. Sürücü veri yolunu tarar, kimlik bilgisini okur ve uygun sürücüyü eşleştirir.
- **Sabit veya eski aygıtlar.** PS/2 denetleyicisi, eski COM port'ları ve VGA gibi aygıtların adresleri platforma göre bilinir veya firmware tarafından bırakılır.
- **Firmware tarafından tanımlanan aygıtlar.** ACPI tabloları veya aygıt ağacı (device tree), hangi aygıtın hangi adreste ve hangi interrupt hattında olduğunu söyler. ARM64 ve RISC-V'de bu yol zorunludur.

PCIe'de her aygıtın bir veri yolu/aygıt/fonksiyon (bus/device/function, BDF) adresi vardır. Aygıtın register'ları, yapılandırma alanındaki BAR (Base Address Register) denen alanlarla fiziksel adres aralıklarına yerleştirilir. Sürücü bu aralıkları kendisi uydurmaz; firmware'in bıraktığı veya kernel'in kaynak dağıtıcısının atadığı değerleri okur. Ayrıntı `pci.md` notunun konusudur.

## Kernel tarafında

### Sürücü modeli ve katmanlar

Modern kernel'lerde sürücü, tek başına bir fonksiyonlar kümesi değil, bir aygıt modeline kayıtlı bir nesnedir. Model genellikle şu katmanlardan oluşur:

```text
uygulama
  system call
dosya sistemi / socket / blok katmanı
  okuma-yazma, kuyruk, tampon
sürücü çekirdeği (driver core)
  aygıt kaydı, sürücü eşleştirme, probe/remove
sürücü
  register erişimi, interrupt, DMA
veri yolu (PCIe, USB, I2C, platform)
aygıt
```

Sürücü çekirdeği, "hangi sürücü hangi aygıta bağlanır" sorusunu yönetir. Bir aygıt bulunduğunda veya bir sürücü yüklendiğinde ikisinin kimlikleri karşılaştırılır. Eşleşme varsa sürücünün `probe` fonksiyonu çağrılır. `probe`, aygıtın gerçekten var olduğunu doğrular, kaynaklarını ayırır ve sürücüyü kullanıma hazır hale getirir. Başarısız olursa ayırdığı her kaynağı geri bırakmalıdır.

Sürücülerin üst katmana sunduğu arayüz aygıt sınıfına göre değişir:

- **Karakter aygıtı:** Bayt akışı sunar. Seri port, klavye, rastgele sayı üreteci tipik örneklerdir. `open`, `read`, `write`, `ioctl`, `poll` gibi işlemler bulunur.
- **Blok aygıtı:** Sabit boyutlu bloklar sunar. Disk ve SSD sürücüleri bu sınıftadır. Üst katman okuma isteklerini sıraya koyar, sürücü de kuyruktan iş alır.
- **Ağ aygıtı:** Paket gönderip alır. Dosya arayüzü yoktur; ağ yığını sürücüye paket bırakır ve sürücüden gelen paketleri alır.

Bazı kernel'lerde bu sınıflar `file_operations`, `block_device_operations` veya `net_device_ops` gibi işlem tablolarıyla temsil edilir. Sürücü, aygıtını bu tabloya kaydeder; üst katman somut aygıtı değil tabloyu görür.

### Sürücü nerede çalışır?

Sürücü iki farklı yerde durabilir:

| Yerleşim | Avantaj | Dezavantaj |
| --- | --- | --- |
| Kernel içinde | Düşük gecikme, doğrudan interrupt ve DMA erişimi, basit çağrı | Hata kernel'i düşürür; güncellemek için yeniden başlatma gerekir |
| Kullanıcı alanında | Hata izole edilir, sürücü yeniden başlatılabilir | Her işlem için IPC gerekir; interrupt ve DMA yönlendirmesi karmaşıktır |

Mikrokernel tasarımlarında sürücülerin çoğu kullanıcı alanındadır. Tek çekirdekli (monolithic) kernel'lerde ise sürücüler kernel içinde çalışır; Linux ve Windows bu modele yakındır. Kullanıcı alanı sürücüsü, aygıt belleğine erişebilmek için yine de kernel'den bir yetki almak zorundadır. Bu yüzden "kullanıcı alanında" olmak, donanıma doğrudan ve denetimsiz erişmek anlamına gelmez; IOMMU ve bellek eşleme kuralları hâlâ devrededir.

Kernel içindeki sürücüler çekirdeğe statik olarak bağlanabilir veya çalışırken yüklenen modüller halinde gelebilir. Modül yükleme, geliştirme sırasında yeniden başlatmayı önler; ancak modülün kernel sürümüyle ABI uyumu, sembol çözümlemesi ve bellek ömrü gibi ek sorunlar getirir.

### Sürücünün yaşam döngüsü

Tipik bir sürücü şu durumlardan geçer:

1. **Kayıt.** Sürücü, desteklediği aygıt kimlikleriyle birlikte çekirdeğe kaydolur.
2. **Eşleştirme.** Bir aygıt bulunduğunda kimlikler karşılaştırılır.
3. **Probe.** Aygıt doğrulanır, BAR'lar ve interrupt hattı alınır, aygıt sıfırlanır ve yapılandırılır.
4. **Çalışma.** Üst katmandan gelen istekler işlenir; interrupt'lar veri akışını sürer.
5. **Askıya alma (suspend).** Aygıt uyku moduna alınmadan önce bekleyen işler durdurulur ve durum kaydedilir.
6. **Devam (resume).** Aygıt yeniden başlatılır ve kaydedilen durum geri yüklenir.
7. **Kaldırma (remove).** Aygıt çıkarılmadan veya sürücü boşaltılmadan önce interrupt'lar kapatılır, DMA durdurulur, kaynaklar bırakılır.

Bu sıra keyfî değildir. Örneğin `probe` sırasında interrupt hattı aygıt sakinleştirilmeden açılırsa, henüz kurulmamış bir veri yapısına interrupt gelebilir. Benzer şekilde `remove` sırasında önce sürücü durumunu serbest bırakıp sonra interrupt'ı kapatmak, handler'ın serbest bırakılmış belleğe erişmesine yol açar.

### Kernel ortamının kısıtları

Sürücü kodu normal bir uygulama gibi yazılamaz:

- Standart C kütüphanesi yoktur. `printf`, `malloc` ve dosya işlemleri yerine kernel'in kendi arayüzleri kullanılır.
- Kernel stack'i küçüktür. Büyük yerel diziler ve derin özyineleme stack taşmasına yol açar.
- Interrupt bağlamında uyunamaz. Uyku, bloklayan bellek ayırma ve kullanıcı belleğine kopyalama bu bağlamda yapılamaz.
- Aynı sürücü aynı anda birden çok CPU'da çalışabilir. Aygıt durumu, kuyruklar ve sayaçlar kilitlenmeden paylaşılamaz.
- Aygıt erişimi her zaman başarılı olmayabilir. Zaman aşımı, beklenmeyen durum ve aygıtın kaybolması normal akışın parçasıdır.
- Kayan nokta kullanımı genellikle yasaktır veya özel kaydetme/geri yükleme gerektirir.

## Implementasyon notları

**Kaynakları doğrulayın, varsaymayın.** `probe` içinde BAR'ın atanmış olduğu, interrupt hattının bağlı olduğu ve aygıtın gerçekten o kimliğe sahip olduğu kontrol edilmelidir. Firmware bazı aygıtları yapılandırmadan bırakabilir. Atanmamış bir BAR'a yazmak, başka bir aygıtın adres alanına zarar verebilir.

**Önce aygıtı susturun, sonra kaynakları bırakın.** Probe sırasında interrupt'lar kapalı olmalı, DMA ve veri yolu ustalığı (bus mastering) kapatılmalıdır. Aygıt yapılandırıldıktan ve sürücü durumu hazır olduktan sonra interrupt açılır. Kaldırma sırasında bu sıra terstir.

**Her bekleme döngüsüne zaman aşımı koyun.** `while (!(inb(port) & HAZIR))` biçimindeki bir döngü, aygıt yoksa veya arızalıysa kernel'i kilitler. Zaman aşımında hatayı üst katmana bildirin ve aygıtı güvenli duruma alın.

**Aygıt durumunu tek bir yerde tutun.** Aynı bilgiyi hem sürücüde hem üst katmanda tutmak, ikisinin ayrışmasına yol açar. Sürücü kendi donanım durumunu, üst katman ise kendi politikasını yönetmelidir.

**Kilidi doğru seviyede tutun.** Aygıtın register'larına erişen kısa kritik bölgeler için spinlock uygundur. Ancak interrupt handler ile process bağlamı aynı kilidi kullanıyorsa, process tarafında yerel interrupt'lar da kapatılmalıdır. Aksi halde process kilidi tutarken interrupt gelir, handler aynı kilidi bekler ve kilitlenme oluşur. Uzun süren işler (DMA tamamlanmasını bekleme, disk okuma) kilit altında yapılmamalıdır.

**DMA adreslerini karıştırmayın.** CPU'nun gördüğü sanal adres ile aygıtın gördüğü veri yolu adresi aynı değildir. IOMMU varsa adres çevirisi yapılır; yoksa fiziksel adres kullanılır. 32 bit DMA sınırı olan aygıtlar, belleğin üst kısmına denk gelen tamponlarla çalışamaz. Bu ayrıntı `dma.md` notunun konusudur.

**Hata yolunu da yazın.** Başarılı `probe` kadar, yarıda kalan `probe` de önemlidir. Alınan her kaynak, hata durumunda ters sırayla bırakılmalıdır. Aksi halde ikinci denemede "kaynak meşgul" hatası alınır ve aygıt bir daha çalışmaz.

**Sürücüyü politika ile doldurmayın.** "Yalnızca root erişebilir", "bu aygıt varsayılan olarak açık gelir" gibi kararlar sürücünün değil, üst katmanın ve yapılandırmanın işidir. Sürücü mekanizmayı sunar.

**Günlük kaydını sınırlayın.** Bozuk bir aygıt saniyede binlerce hata üretebilir. Her hatada seri port'a yazmak, hatanın kendisinden daha büyük bir performans sorununa dönüşür. Oran sınırlaması veya sayaç kullanılmalıdır.

## Mimariye göre farklar

| Mimari | Aygıt keşfi | Erişim | Interrupt denetleyicisi |
| --- | --- | --- | --- |
| x86-64 | PCIe, ACPI, eski sabit adresler | Port I/O ve MMIO | PIC, APIC, MSI/MSI-X |
| ARM64 | Aygıt ağacı veya ACPI | Yalnızca MMIO | GIC |
| RISC-V | Aygıt ağacı veya ACPI | Yalnızca MMIO | PLIC veya AIA |

ARM64 ve RISC-V'de port I/O yoktur. Bu mimarilerde sürücünün register erişimi tamamen MMIO üzerinden yapılır. Aygıt adreslerinin sabit olduğu varsayılmamalı; firmware'in verdiği tablolardan okunmalıdır. x86'da bile modern aygıtların çoğu MMIO kullanır; port I/O daha çok eski ve basit aygıtlarda görülür.

## Sık yapılan hatalar

- `probe` içinde interrupt'ı aygıt hazır olmadan açmak.
- Aygıtın varlığını doğrulamadan register'larına yazmak.
- Zaman aşımı olmadan donanım bitini beklemek.
- Aynı sürücü verisini farklı CPU'larda kilitsiz paylaşmak.
- Interrupt handler içinde uyumak veya bloklayan bellek ayırmak.
- `remove` sırasında önce sürücü belleğini serbest bırakıp sonra interrupt'ı kapatmak.
- DMA için sanal adresi doğrudan aygıta vermek.
- Hata durumunda yarıda kalan kaynakları geri bırakmamak.
- Aygıt adreslerini firmware tabloları yerine sabit olarak gömmek.
- Sürücüye dosya izinleri veya kullanıcı politikası koymak.

## İlgili notlar

- [Donanım erişimi: port I/O ve MMIO](donanim-erisimi.md)
- [Kesme tabanlı sürücüler ve interrupt handler tasarımı](kesme-tabanli-suruculer.md)
- [Seri port (UART)](uart.md)
- [RTC ve CMOS](rtc.md)
- [PIC (8259)](../04-kernel/pic.md)
- [IDT (Interrupt Descriptor Table)](../04-kernel/idt.md)

## Kaynaklar

- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 3A, Bölüm 6 — Interrupt and Exception Handling
- PCI-SIG, *PCI Local Bus Specification*, Revizyon 3.0 — Configuration Space ve BAR'lar
- PCI-SIG, *PCI Express Base Specification* — Configuration Space, MSI/MSI-X
- ACPI Specification, Bölüm 6 — Device Configuration; <https://uefi.org/specifications> (erişim: 29 Ekim 2026)
- Devicetree Specification; <https://www.devicetree.org/specifications/> (erişim: 29 Ekim 2026)
- Linux kernel dokümantasyonu, "Driver Model"; <https://docs.kernel.org/driver-api/driver-model/> (erişim: 29 Ekim 2026)
- Linux kernel dokümantasyonu, "Everything you never wanted to know about kobjects, ksets, and ktypes"; <https://docs.kernel.org/core-api/kobject.html> (erişim: 29 Ekim 2026)
- OSDev Wiki, "PCI"; <https://wiki.osdev.org/PCI> (erişim: 29 Ekim 2026)
