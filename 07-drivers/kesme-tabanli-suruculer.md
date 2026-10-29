# Kesme tabanlı sürücüler ve interrupt handler tasarımı

Bir aygıtın ne zaman veri üreteceğini veya işini bitireceğini önceden bilemeyiz. Sürücü sürekli durum register'ını okuyarak bekleyebilir (polling), ama bu CPU'yu boşa harcar ve gecikmeyi aygıtın hızına bağlar. Interrupt, aygıtın hazır olduğunda işlemciyi kendiliğinden uyarmasını sağlar. Sürücü açısından asıl zorluk interrupt'ı açmak değil, o interrupt geldiğinde kernel'in tutarlılığını bozmadan mümkün olan en kısa sürede tepki verip geri kalan işi güvenli bir bağlama devretmektir.

Interrupt handler, kernel'in en kısıtlı bağlamında çalışır. Uyuyamaz, uzun süre bekleyemez ve aynı anda başka bir CPU'da çalışan kodla aynı veriye erişebilir. Bu yüzden iyi bir handler'ın ölçüsü "ne kadar çok iş yaptığı" değil, "ne kadar az iş yapıp gerisini doğru yere bıraktığıdır".

## Ön koşullar

- [IDT (Interrupt Descriptor Table)](../04-kernel/idt.md) — giriş çerçevesi, gate tipleri, `iretq`
- [PIC (8259)](../04-kernel/pic.md) — IRQ yönlendirmesi ve EOI
- [Donanım erişimi: port I/O ve MMIO](donanim-erisimi.md)
- [Sürücü nedir, kernel içinde nerede durur](surucu-nedir.md)

## Kapsam

Bu not x86-64'ü, PIC/APIC yolunu ve PCI MSI/MSI-X'i esas alır. Handler tasarımının ilkeleri mimariden bağımsızdır; ARM64 GIC ve RISC-V PLIC farkları ayrıca belirtilir. Kod örnekleri C ile yazılmıştır ve gerçek bir kernel API'si değil, tasarımı göstermek için kısaltılmıştır.

## Problem

Polling ile çalışan bir sürücü şöyle görünür:

```c
while (!(inb(UART + 5) & 0x01))
    ;   /* veri gelene kadar CPU burada döner */
return inb(UART);
```

Bu döngü, veri gelene kadar CPU'yu meşgul eder. Aynı CPU'da başka bir process çalışamaz; güç tüketimi artar. Veri seyrek geliyorsa boşa harcanan süre çok büyüktür. Interrupt bu sorunu çözer: sürücü beklemeyi bırakır, aygıt hazır olduğunda işlemciye haber verir.

Ancak interrupt, yeni bir sorun getirir. Handler, çalışan her ne varsa onu böler. O anda kernel bir kilit tutuyor, bir veri yapısını güncelliyor veya başka bir aygıtla konuşuyor olabilir. Handler'ın bu durumu bozmaması, aygıtı doğru sırayla onaylaması ve uzun işleri interrupt dışına taşıması gerekir. Aksi halde polling'den daha kötü sonuçlar alınır: kaybolan interrupt'lar, kilitlenmeler, interrupt fırtınası ve veri bozulması.

## Donanım seviyesinde

### Interrupt nasıl oluşur

Bir aygıt interrupt üretmek için üç yoldan birini kullanır:

- **Hat tabanlı (legacy).** Aygıt, PIC veya IOAPIC'e bağlı bir IRQ hattını yükseltir. Denetleyici bunu bir vektör numarasına çevirip işlemciye iletir.
- **MSI (Message Signaled Interrupt).** Aygıt, belirli bir adrese özel bir veri yazarak interrupt'ı "mesaj" olarak gönderir. Fiziksel hat yoktur.
- **MSI-X.** MSI'ın genişletilmiş biçimidir; aygıt birden çok vektör ve her vektör için ayrı hedef adres kullanabilir. Modern PCIe aygıtları genellikle bunu tercih eder.

Hat tabanlı interrupt'larda iki elektriksel davranış vardır:

| Tür | Davranış | Sürücü için sonucu |
| --- | --- | --- |
| Level-triggered | Hat, aygıt onaylayana kadar yüksek kalır | Kaynak temizlenmeden EOI gönderilirse interrupt tekrar gelir |
| Edge-triggered | Hattın yükselen kenarı olayı tetikler | Hat seviyesi önemli değildir; olay kenarda yakalanır |

Level-triggered bir hat, aygıtın interrupt durum register'ı temizlenene kadar asserted kalır. Sürücü önce aygıttaki kaynağı temizlemeli, sonra kesme denetleyicisine EOI göndermelidir. Sıra ters olursa denetleyici EOI'den hemen sonra hattı hâlâ yüksek görür ve aynı interrupt'ı yeniden teslim eder. Bu, saniyede binlerce interrupt üreten bir fırtınaya dönüşebilir.

Edge-triggered hatlarda ise aygıt, sürücü hattı maskelediği sırada yeni bir olay üretirse bu olay kaybolabilir. Bu yüzden sürücü, interrupt'ı yeniden açmadan önce aygıt durumunu kontrol etmelidir.

### PIC, APIC ve vektör

Eski x86 sistemlerinde iki 8259 PIC, IRQ'ları toplayıp vektör numarasına çevirir. Modern sistemlerde IOAPIC, aygıt hatlarını yerel APIC'e yönlendirir; yerel APIC de vektörü işlemciye teslim eder. MSI/MSI-X'te aygıt doğrudan yerel APIC'e mesaj gönderir.

Handler'ın görevi açısından bu ayrıntıların çoğu alt katmanda kalır. Sürücünün bilmesi gerekenler şunlardır:

- Hangi vektörün kendisine ait olduğu.
- Interrupt'ın paylaşılıp paylaşılmadığı.
- Aygıtı hangi register yazımıyla susturacağı.
- EOI'nin kime ve hangi sırayla gönderileceği.

Vektör numarası ile IRQ numarası aynı şey değildir. PIC yeniden eşlendiğinde IRQ 0–7 vektör 32–39'a, IRQ 8–15 vektör 40–47'ye düşer. Sürücü IRQ numarasını bilir; vektör eşlemesini kesme denetleyicisi katmanı yapar.

### Paylaşılan interrupt

PCI hat tabanlı interrupt'lar (INTx) birden çok aygıt tarafından paylaşılabilir. Bu durumda aynı vektör için birden çok handler çalışır. Her handler önce interrupt'ın kendi aygıtından gelip gelmediğini kontrol etmelidir. Kendi aygıtı değilse "bu benim değildi" diyerek geri dönmeli, başka bir aygıtın durum register'ını temizlememelidir.

MSI ve MSI-X normalde paylaşılmaz; her vektör belirli bir aygıta aittir. Yine de yazılım aynı vektörü birden çok aygıta atayabilir. Bu yüzden "MSI var, paylaşım yok" varsayımı yerine kayıt sırasında bunun garanti edilip edilmediği bilinmelidir.

### Sahte interrupt

Denetleyici, gerçek bir istek olmadan da interrupt teslim edebilir. PIC'te IRQ 7 ve IRQ 15 sahte interrupt olarak bilinir; APIC'te de spurious interrupt vektörü vardır. Sahte IRQ 7 için EOI gönderilmez. Sahte IRQ 15 ise slave'ten gelmediği için yalnızca master'a EOI gerektirir. APIC'in spurious vektöründe de EOI gönderilmez. Bu ayrım yapılmazsa gerçekten bekleyen bir interrupt'ın onayı kaybolabilir. Kernel, sahte interrupt'ları saymalı ve belirli bir eşiği aşınca donanımı veya yapılandırmayı sorgulamalıdır.

## Kernel tarafında

### Üst yarı ve alt yarı

Interrupt handler'ı iki parçaya ayırmak yaygın bir tasarımdır:

| Parça | Nerede çalışır | Ne yapar | Kısıt |
| --- | --- | --- | --- |
| Üst yarı (hard IRQ) | Interrupt bağlamı | Durumu okur, aygıtı onaylar, işi kuyruğa koyar | Uyuyamaz, uzun süremez |
| Alt yarı (bottom half) | Kernel thread'i, softirq veya workqueue | Veriyi işler, üst katmana teslim eder | Uyuyabilir veya daha uzun çalışabilir |

Üst yarı mümkün olduğunca kısa olmalıdır. Aygıtın "neden interrupt ürettiğini" öğrenmek, durum register'ını okumak, gerekli bayrakları temizlemek ve veriyi bir halka tampona (ring buffer) bırakmak çoğu zaman yeterlidir. Paket işleme, dosya sistemi güncellemesi veya kullanıcı belleğine kopyalama alt yarıya bırakılır.

Alt yarı için seçenekler:

- **Softirq / tasklet.** Interrupt dönüşünde hızlıca çalışır, uyuyamaz. Ağ ve blok katmanlarının sık kullandığı yol.
- **Workqueue.** Kernel thread'i üzerinde çalışır, uyuyabilir. Yavaş aygıtlar ve uzun işler için uygundur.
- **Threaded IRQ.** Interrupt'ın büyük kısmı doğrudan bir kernel thread'ine devredilir; üst yarı yalnızca aygıtı susturur ve thread'i uyandırır. USB HID ve I2C gibi yavaş veri yollarında yaygındır.

Hangi seçeneğin kullanılacağı, işin uyuyup uyuyamayacağına ve gecikme gereksinimine bağlıdır. Her interrupt'ta thread uyandırmak, çok sık gelen olaylarda context switch maliyeti getirir. Buna karşılık her şeyi üst yarıda yapmak, interrupt gecikmesini ve kayıp riskini büyütür.

### Handler kaydı ve sözleşmesi

Sürücü, interrupt hattını bir handler'a bağlar. Tipik bir kayıt arayüzü şöyle görünür:

```c
enum {
    IRQ_ISLENDI = 1,
    IRQ_BENIM_DEGIL = 0,
};

typedef int (*irq_handler_t)(unsigned int irq, void *baglam);

int irq_kaydet(unsigned int irq, irq_handler_t handler,
               unsigned long bayraklar, const char *ad, void *baglam);
void irq_birak(unsigned int irq, void *baglam);
```

Handler'ın dönüş değeri, interrupt'ın gerçekten bu aygıttan gelip gelmediğini bildirir. Paylaşılan hatlarda kernel, tüm handler'ları çağırır ve en az biri "işledim" derse interrupt'ı tüketilmiş sayar. Hiçbiri sahiplenmezse bu bir sahte interrupt olarak sayılır.

Kayıt sırasında bayraklar önemlidir:

- **Paylaşımlı hat.** Handler'ın kendi aygıtını kontrol etmesi zorunludur.
- **Level/edge bilgisi.** Denetleyicinin doğru programlanması için gereklidir.
- **IRQ devre dışı bırakılarak mı çalışsın.** Handler'ın kendi aygıtının interrupt'ını yeniden açma sorumluluğunu değiştirir.
- **Tek seferlik mi.** Bazı aygıtlar yalnızca bir kez interrupt üretir; kayıt sonrası otomatik olarak bırakılmalıdır.

### Tipik bir üst yarı

```c
static struct halka klavye_kuyrugu;

static int klavye_irq(unsigned int irq, void *baglam)
{
    uint8_t durum = inb(0x64);

    /* Çıktı tamponunda veri yoksa interrupt başka bir kaynaktan gelmiştir. */
    if (!(durum & 0x01))
        return IRQ_BENIM_DEGIL;

    uint8_t tarama_kodu = inb(0x60);

    if (!halka_yaz(&klavye_kuyrugu, tarama_kodu)) {
        /* Tampon dolu: veri kaybını önlemek için aygıtı geçici olarak sustur. */
        klavye_kesmeyi_kapat();
        klavye_kuyrugu.tasti = 1;
    }

    return IRQ_ISLENDI;
}
```

Bu örnekte üst yarı yalnızca durumu okur, veriyi kuyruğa koyar ve gerekirse akış kontrolü yapar. Tarama kodunu Unicode karaktere çevirmek, ölü tuşları işlemek ve kullanıcıya iletmek alt yarıya aittir. PS/2 ayrıntıları `ps2.md` notunun konusudur.

### Halka tampon ve eşzamanlılık

Interrupt üreticisi ile tüketici farklı bağlamlarda çalışır. Üretici handler, tüketici ise process veya kernel thread'idir. Tek üretici ve tek tüketici varsa kilit yerine dikkatli bir baş/kuyruk protokolü kullanılabilir:

```c
struct halka {
    uint8_t veri[256];
    volatile uint32_t yaz;   /* yalnızca üretici artırır */
    volatile uint32_t oku;   /* yalnızca tüketici artırır */
    volatile int tasti;
};
```

Üretici, `yaz` indeksini güncellemeden önce veriyi yazar. Tüketici, veriyi okumadan önce `yaz` indeksini okur. Bu sıralama derleyici ve işlemci tarafından korunmalıdır; aksi halde tüketici, henüz yazılmamış bir slotu okur. Tek üretici/tek tüketici halkasında bile indekslerin atomik güncellenmesi ve bellek barrier'ı gerekir.

Birden çok üretici veya tüketici varsa kilit gerekir. Kilit seçerken hangi bağlamlarda alındığına bakılmalıdır:

- Yalnızca process bağlamında alınıyorsa mutex yeterli olabilir.
- Interrupt handler da alıyorsa mutex kullanılamaz; spinlock gerekir.
- Process bağlamı kilidi tutarken aynı CPU'da interrupt gelirse, handler kilidi beklerken kilitlenme oluşur. Bu yüzden process tarafında `spin_lock_irqsave` benzeri bir yöntemle yerel interrupt'lar kapatılmalıdır.

### EOI sırası

Level-triggered bir aygıtta doğru sıra genellikle şöyledir:

1. Aygıtın interrupt durum register'ını oku.
2. Hangi olayın geldiğini belirle ve gerekli veriyi al.
3. Aygıttaki interrupt kaynağını temizle (veya maskeyi kaldır).
4. Kesme denetleyicisine EOI gönder.
5. Gerekliyse alt yarıyı uyandır.

Aygıt kaynağı temizlenmeden EOI gönderilirse hat asserted kalır ve denetleyici interrupt'ı yeniden teslim eder. Bu yüzden "önce EOI, sonra temizle" sırası level-triggered sistemlerde yanlıştır.

PIC'te slave'den gelen bir IRQ için EOI hem slave'e hem master'a gönderilir. APIC'te EOI, yerel APIC'in EOI register'ına yazılır. MSI'da da teslim yerel APIC üzerinden olduğu için EOI gerekir; ancak hat paylaşımı olmadığı için "başka aygıt mı" kontrolü genellikle gerekmez.

### Interrupt'ı yeniden açma yarışı

Aygıtı susturup kuyruğu boşalttıktan sonra interrupt'ı yeniden açmak, dikkat gerektiren bir yarıştır:

```text
1. Aygıt interrupt'ını kapat
2. Kuyruktaki veriyi işle
3. Aygıt durumunu tekrar kontrol et
4. Interrupt'ı aç
5. Durumu bir kez daha kontrol et
```

Üçüncü adım, kapalıyken gelen olayı yakalar. Dördüncü adımdan sonra beşinci adımı yapmak, "interrupt'ı açtıktan hemen sonra ama henüz interrupt gelmeden" oluşan olayı yakalar. Bu iki kontrol olmadan, özellikle edge-triggered aygıtlarda veri kaybolabilir.

### Interrupt bağlamının kuralları

Interrupt handler içinde şunlar yapılmamalıdır:

- Uyku gerektiren kilit almak (mutex, semafor).
- Bloklayan bellek ayırmak.
- Kullanıcı belleğine doğrudan kopyalamak; bu işlem page fault üretebilir ve uyku gerektirebilir.
- Uzun süreli döngülerde beklemek.
- Sınırsız log yazmak.
- Aynı CPU'da tekrar interrupt beklemek.

Interrupt gate kullanıldığında girişte RFLAGS.IF temizlenir; yani handler, kendisi açmadıkça aynı CPU'da maskelenebilir interrupt'ları almaz. Bu, handler'ı kısa tutmayı kolaylaştırır ama uzun süren bir handler'ın sistem gecikmesini büyüttüğü gerçeğini değiştirmez. Farklı CPU'larda interrupt'lar gelmeye devam eder; handler'ın paylaştığı veri hâlâ yarış durumuna açıktır.

## Implementasyon notları

**Önce durumu oku, sonra onayla.** Aygıtın neden interrupt ürettiğini bilmeden EOI göndermek, olayı kaybetmenin en yaygın yoludur. Durum register'ını okuyup hangi bitin set olduğunu belirleyin; yalnızca kendi bitinizi temizleyin.

**Paylaşılan hatlarda kendi aygıtınızı doğrulayın.** Başka bir aygıtın interrupt'ını "işledim" diye sahiplenmek, o aygıtın olayının kaybolmasına yol açar. Kendi durum register'ınız temizse `IRQ_BENIM_DEGIL` döndürün.

**Aygıtı doğru yerde susturun.** Tampon dolduğunda interrupt'ı kapatmak, veri kaybını önlemenin yaygın yoludur. Ancak kapatma işlemi aygıtın kendi maske register'ı üzerinden yapılmalı; global `cli` kullanmak diğer aygıtları da etkiler.

**Alt yarıyı sınırlayın.** Her interrupt'ta bir thread uyandırmak, yüksek hızlı aygıtlarda context switch fırtınası yaratır. Toplu işleme (batch), interrupt birleştirme (coalescing) veya NAPI benzeri bir "önce polling, sonra interrupt" geçişi düşünülebilir.

**Kilit sırasını belgeleyin.** Birden çok kilit kullanılıyorsa hangi sırayla alınacağı yazılmalıdır. Interrupt handler'ın aldığı kilit, process bağlamında alınan kilitten önce geliyorsa ters sıra kilitlenme üretir.

**DMA tamamlanmasını doğru sıralayın.** Aygıt, DMA yazmasını bitirdiğinde interrupt üretir. Handler, DMA tamponundaki veriyi okumadan önce gerekli okuma barrier'ını koymalıdır. Aksi halde önbellek tutarlılığı olmayan platformlarda eski veri görülebilir. DMA ve önbellek ayrıntıları `dma.md` notundadır.

**Interrupt fırtınasını önleyin.** Bozuk bir aygıt sürekli interrupt üretiyorsa handler her seferinde log yazmamalıdır. Sayaç, oran sınırlaması ve belirli bir eşikten sonra aygıtı devre dışı bırakma mekanizması gerekir. Aksi halde sistem canlı kalır ama hiçbir iş yapamaz.

**Sahte interrupt'ı ayırt edin.** Sahte interrupt'ta EOI göndermemek gerekir. PIC'te IRQ 7/15, APIC'te spurious vektör için ayrı yol izlenmelidir. Sahte interrupt sayısı tanılama için değerli bir göstergedir.

**Handler'ı test edilebilir tutun.** Aygıt erişimini küçük bir soyutlama arkasına almak, handler'ı emülatörde ve birim testinde çalıştırmayı kolaylaştırır. Ancak soyutlamanın kendisi interrupt yoluna ek maliyet getirmemelidir.

**MSI/MSI-X'i erken düşünün.** Çok kuyruklu ağ ve depolama aygıtlarında her kuyruğa ayrı vektör vermek, tek vektörde sıraya girmekten çok daha iyi ölçeklenir. MSI-X tablosunun BAR içindeki konumu ve vektör başına maskeleme desteği aygıt spesifikasyonundan okunmalıdır.

## Mimariye göre farklar

| Mimari | Denetleyici | Vektörleme | EOI |
| --- | --- | --- | --- |
| x86-64 | PIC, IOAPIC + yerel APIC, MSI/MSI-X | Vektör numarası IDT girdisini seçer | PIC komut portu veya APIC EOI register'ı |
| ARM64 | GIC | GIC interrupt ID'si, VBAR_EL1'deki grup girişine dallanma | GIC'e EOIR yazılır |
| RISC-V | PLIC veya AIA | `scause` içindeki neden kodu; `stvec` girişi | PLIC claim/complete protokolü |

ARM64'te GIC, interrupt'ı önce "claim" ettirir; handler işini bitirince aynı ID ile "complete" yazar. Bu, x86'daki EOI'ye benzer bir sorumluluktur ama ayrı bir adım olarak görünür. RISC-V PLIC'te de claim/complete vardır; yanlış sıra, interrupt'ın bir daha teslim edilmemesine yol açar.

## Sık yapılan hatalar

- Aygıt kaynağını temizlemeden EOI göndermek; level-triggered hatta interrupt fırtınası.
- Paylaşılan hatlarda kendi aygıtını kontrol etmeden "işledim" demek.
- Sahte interrupt'ta EOI göndermek.
- Handler içinde uyumak veya bloklayan kilit almak.
- Aynı veriye farklı CPU'larda kilitsiz erişmek.
- Interrupt'ı yeniden açtıktan sonra durumu bir kez daha kontrol etmemek.
- Tampon dolduğunda veri kaybını sessizce kabul etmek.
- Her interrupt'ta ağır log yazmak.
- DMA tamponunu barrier olmadan okumak.
- Alt yarıyı hiç kullanmayıp tüm işi üst yarıda yapmak.

## İlgili notlar

- [IDT (Interrupt Descriptor Table)](../04-kernel/idt.md)
- [PIC (8259)](../04-kernel/pic.md)
- [Sürücü nedir, kernel içinde nerede durur](surucu-nedir.md)
- [Donanım erişimi: port I/O ve MMIO](donanim-erisimi.md)
- [Seri port (UART)](uart.md)
- [RTC ve CMOS](rtc.md)
- DMA: `dma.md`

## Kaynaklar

- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 3A, Bölüm 6 — Interrupt and Exception Handling
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 3A, Bölüm 10 — Advanced Programmable Interrupt Controller (APIC)
- Intel 8259A Programmable Interrupt Controller veri sayfası — EOI ve sahte interrupt davranışı
- PCI-SIG, *PCI Local Bus Specification*, Revizyon 3.0 — Interrupt pinleri ve paylaşım
- PCI-SIG, *PCI Express Base Specification* — MSI ve MSI-X
- Arm® Architecture Reference Manual for A-profile architecture, Bölüm D1 ve GIC spesifikasyonu — interrupt girişi, claim/complete
- The RISC-V Instruction Set Manual, Volume II: Privileged Architecture — `scause`, `stvec`; PLIC spesifikasyonu
- Linux kernel dokümantasyonu, "Linux generic IRQ handling"; <https://docs.kernel.org/core-api/genericirq.html> (erişim: 29 Ekim 2026)
- Linux kernel dokümantasyonu, "PCI MSI/MSI-X"; <https://docs.kernel.org/PCI/msi-howto.html> (erişim: 29 Ekim 2026)
- OSDev Wiki, "Interrupts"; <https://wiki.osdev.org/Interrupts> (erişim: 29 Ekim 2026)
