# Donanım erişimi: port I/O ve MMIO

İşlemci bir aygıtın register'ına iki farklı adres alanı üzerinden ulaşır. Port I/O'da aygıt, bellekten ayrı 16 bitlik bir adres alanında durur ve yalnızca `IN`/`OUT` komutlarıyla erişilir. MMIO'da (Memory-Mapped I/O) aygıt register'ları fiziksel bellek adres alanına yerleştirilir ve sıradan yükleme/kaydetme komutlarıyla okunup yazılır. Bu not, iki yöntemin sürücü tarafından nasıl kullanıldığını, erişim sırasının neden önemli olduğunu ve gerçek bir implementasyonda hangi tuzakların bulunduğunu anlatır.

## Ön koşullar

- [MMIO ve port I/O](../01-mimari/mmio-ve-port-io.md) — iki yöntemin temel tanımı
- [Sürücü nedir, kernel içinde nerede durur](surucu-nedir.md)
- [Paging](../05-memory/) — fiziksel adreslerin sanal adres alanına eşlenmesi
- [Endianness](../01-mimari/endianness.md)

## Kapsam

Bu not x86-64'ü ve long mode'u esas alır; port I/O yalnızca x86 ailesinde bulunduğu için ARM64 ve RISC-V karşılaştırması kısa tutulmuştur. Örnekler C ve GNU inline assembly ile yazılmıştır. `outb`, `inb` ve sayfa tablosu yardımcılarının tanımlı olduğu varsayılır.

## Problem

Bir sürücünün aygıtla konuşması için üç soruya cevap vermesi gerekir: register hangi adreste, hangi genişlikte okunacak ve işlemcinin bu erişimi ne zaman tamamlandığı nereden bilinecek?

Adresi bilmek tek başına yetmez. Aygıt register'ı normal bellek gibi önbelleğe alınırsa işlemci yazılan değeri önbellekte tutabilir ve aygıta hiç göndermeyebilir. Derleyici `volatile` olmayan bir erişimi tamamen silebilir veya iki erişimi birleştirebilir. Aygıt 32 bitlik bir register beklerken 8 bitlik erişim yapılırsa bazı aygıtlar bunu reddeder veya yanlış yorumlar. Üstelik yazma işlemi veri yolundaki bir köprüde bekleyebilir; sürücü "yazdım" dediğinde aygıt henüz görmemiş olabilir.

Bu yüzden donanım erişimi, "adrese değer yaz" işleminden daha fazlasıdır: doğru adres alanı, doğru genişlik, doğru önbellek türü ve doğru sıralama birlikte sağlanmalıdır.

## Donanım seviyesinde

### Port I/O

x86'da bellekten bağımsız, 65.536 portluk bir I/O adres alanı vardır. Erişim `IN` ve `OUT` komutlarıyla yapılır. Port numarası 255'ten küçükse komutun içinde doğrudan verilebilir; daha büyük numaralar `DX` register'ı üzerinden geçirilir. Veri `AL`, `AX` veya `EAX` üzerinde taşınır. Bu, tek bayt, 16 bit ve 32 bit erişimlerin aynı komut ailesiyle yapılabildiği anlamına gelir.

Blok transfer için `INS` ve `OUTS` komutları vardır. Bunlar `DX`'teki port ile bellek arasında `REP` önekiyle birden çok birim taşır. Kaynak adres `DS:SI`, hedef adres `ES:DI` üzerinden gelir. Bu komutlar özellikle eski ATA denetleyicilerinde sektör okumak için kullanılır.

Port I/O'nun önemli özellikleri:

- Adres alanı bellekten ayrıdır. Aynı sayısal değer, bir port ve bir bellek adresi olarak farklı yerlere gider.
- Paging bu alanı etkilemez. Port numarası sanal adrese çevrilmez.
- Kullanıcı modunda `IN`/`OUT` varsayılan olarak çalışmaz. İzin, RFLAGS'teki IOPL alanına ve TSS'teki I/O izin bitmap'ine bağlıdır.
- Sanal makinede hipervizör, I/O bitmap'iyle bu erişimleri yakalayabilir.

Basit bayt erişimi şöyle yapılır:

```c
#include <stdint.h>

static inline void outb(uint16_t port, uint8_t deger)
{
    __asm__ volatile ("outb %0, %1" : : "a"(deger), "Nd"(port));
}

static inline uint8_t inb(uint16_t port)
{
    uint8_t deger;
    __asm__ volatile ("inb %1, %0" : "=a"(deger) : "Nd"(port));
    return deger;
}
```

`"Nd"` kısıtı, derleyicinin port numarasını uygun olduğunda doğrudan komut içine, olmadığında `DX`'e koymasını sağlar. `volatile` burada erişimin silinmesini engeller; ancak derleyiciye ve işlemciye sıralama garantisi vermez.

Eski aygıtlarda iki `OUT` arasına kısa bir gecikme gerekebilir. Yaygın yöntem, kullanılmayan `0x80` port'una bir bayt yazmaktır. Bu, gerçek bir zamanlayıcı değil, veri yoluna küçük bir bekleme eklemenin taşınabilir olmayan ama yaygın yoludur.

### MMIO

MMIO'da aygıt register'ları fiziksel adres alanının bir parçası gibi görünür. O adrese yapılan okuma veya yazma RAM'e değil aygıta gider. Erişim için `MOV` gibi sıradan komutlar kullanılır. Modern aygıtların çoğu — APIC, PCIe aygıtları, framebuffer, NVMe denetleyicisi — MMIO kullanır.

MMIO'nun port I/O'ya göre avantajı, aynı komut setinin kullanılabilmesi ve geniş adres alanına yayılabilmesidir. Dezavantajı, bu adreslerin sayfa tablolarında özel bir önbellek türüyle eşlenmesi gerektiğidir. Normal önbelleğe alınabilir bellek olarak eşlenen bir aygıt register'ı, işlemcinin önbelleğinde kalır ve aygıt beklenen değeri hiç görmez.

x86-64'te bu, sayfa tablosu girdisindeki PCD (Page Cache Disable) ve PWT (Page Write-Through) bayrakları veya PAT (Page Attribute Table) ile ayarlanır. ARM64'te MAIR_EL1 içindeki Device bellek türü, RISC-V'de platformun bellek nitelikleri ve Svpbmt eklentisi devreye girer.

### BAR'lar ve kaynak keşfi

PCIe aygıtının register'ları, yapılandırma alanındaki BAR'larla fiziksel adres aralıklarına yerleştirilir. Her BAR bir taban adresi ve bir boyut taşır. Sürücü BAR'ı okur, türünü (bellek veya I/O) belirler ve o aralığı kendi adres alanına eşler.

BAR boyutu, tüm bitleri 1 yazıp geri okuyarak bulunur. Aygıtın yazma dekoderi kapatılmalı, eski değer saklanmalı ve işlem sonunda geri yüklenmelidir. 64 bit bellek BAR'ı iki ardışık BAR slotu kullanır. Bu ayrıntılar `pci.md` notunun konusudur; burada önemli olan, sürücünün adresi sabit varsaymamasıdır.

### Erişim genişliği ve yan etkiler

Aygıt register'ları genellikle sabit genişlikte erişim bekler. 32 bitlik bir register'a 8 bitlik yazma yapmak, bazı aygıtlarda hataya, bazılarında ise sessizce yanlış davranışa yol açar. 64 bitlik bir register'ı iki 32 bitlik parça halinde okumak, arada değer değişirse tutarsız bir sonuç verebilir.

Bazı register'lar okunduğunda yan etki üretir: interrupt durum register'ı okunduğunda temizlenir, FIFO register'ı okunduğunda kuyruktan bir eleman çekilir. Bazı bayraklar "1 yazınca temizle" (write-one-to-clear) biçimindedir. Bu yüzden register'a kör bir read-modify-write yapmak tehlikelidir. Aygıtın veri sayfası, hangi register'ın nasıl erişileceğini söyler.

### Sıralama ve görünürlük

Donanım erişiminde üç ayrı sıralama katmanı vardır:

1. **Derleyici sıralaması.** `volatile` erişimlerin silinmesini ve birleştirilmesini engeller, ancak diğer erişimlere göre sırasını tam olarak garanti etmez.
2. **İşlemci sıralaması.** x86 güçlü bir bellek modeline sahiptir; yine de MMIO ve normal bellek arasında barrier gerekebilir. ARM64 ve RISC-V'de sıralama daha zayıftır.
3. **Veri yolu sıralaması.** PCIe yazmaları "posted" olabilir; işlemci komutu tamamladığında yazma henüz köprüde veya aygıtın girişinde olabilir.

Pratikte sürücü şu kurallara uyar:

- Kontrol register'ına yazdıktan sonra aygıtın bunu gördüğünden emin olmak için aynı register'ı geri okuyun veya uygun bir barrier kullanın.
- DMA tanımlayıcısını belleğe yazdıktan sonra aygıta "kapıyı çal" (doorbell) yazmadan önce yazma barrier'ı koyun. Aksi halde aygıt, henüz belleğe yazılmamış tanımlayıcıyı okuyabilir.
- Interrupt durum register'ını okuduktan sonra, aygıtın yeni durumunu görmeden önce okuma barrier'ı koyun.

x86'da `mfence`, `lfence` ve `sfence`; ARM64'te `dmb`, `dsb` ve `isb`; RISC-V'de `fence` komutları bu işi yapar. Derleyici tarafında `asm volatile ("" ::: "memory")` bir derleyici barrier'ıdır; donanım barrier'ının yerini tutmaz.

## Kernel tarafında

### Kaynağı alma ve eşleme

Sürücü önce aygıtın fiziksel adres aralığını ve interrupt numarasını öğrenir. Bu bilgi PCI BAR'ından, ACPI `_CRS` yönteminden veya aygıt ağacındaki `reg` özelliğinden gelir. Ardından aralık, kernel'in sanal adres alanına eşlenir:

```c
/* Fiziksel aygıt aralığını uncacheable olarak eşler.
   Gerçek kernelde hata döndürür ve boyut sayfa sınırına yuvarlanır. */
void *mmio_esle(uintptr_t fiziksel, size_t boyut);
```

Eşleme yapılırken şunlar sağlanmalıdır:

- Aralık, aygıtın beklediği önbellek türüyle eşlenir. Kontrol register'ları için uncacheable; framebuffer gibi büyük ve sıralı yazılan alanlar için write-combining düşünülebilir.
- Eşleme yalnızca aygıtın sahip olduğu aralığı kapsar. Tüm fiziksel belleği veya komşu aygıtların BAR'larını eşlemek gereksiz ve tehlikelidir.
- Aynı aralık iki sürücü tarafından talep edilmemelidir. Kernel'in kaynak kayıt mekanizması (Linux'ta `request_mem_region`) bunu engeller.
- Eşlenen adres normal `kmalloc` belleği gibi `free` edilmez; ayrı bir `iounmap` gerekir.

### Erişim yardımcıları

```c
static inline uint32_t mmio_oku32(uintptr_t adres)
{
    uint32_t deger = *(volatile uint32_t *)adres;
    __asm__ volatile ("" ::: "memory");   /* sonraki erişimler öne geçmesin */
    return deger;
}

static inline void mmio_yaz32(uintptr_t adres, uint32_t deger)
{
    __asm__ volatile ("" ::: "memory");   /* önceki erişimler geride kalmasın */
    *(volatile uint32_t *)adres = deger;
}
```

Bu yardımcılar derleyici sıralamasını düzeltir; işlemci ve veri yolu sıralaması için gerekiyorsa ayrıca `mfence` veya aygıta özgü okuma-geri-yazma eklenmelidir. Linux'un `readl`/`writel` ailesi bu işleri mimariye göre yapar; kendi kernel'inizde benzer bir soyutlama kurmak, her sürücüde çıplak işaretçi kullanmaktan daha güvenlidir.

MMIO alanına `memcpy` veya `memset` çağırmak doğru değildir. Bu fonksiyonlar erişim genişliğini ve sırasını aygıtın beklediği gibi korumaz; geniş vektör komutları veya hizalanmamış erişimler üretebilir. Bunun yerine genişliği belli yardımcılar veya `memcpy_toio` benzeri özel fonksiyonlar kullanılır.

### Port I/O ve MMIO birlikte

Bir sürücü aynı aygıtta iki yöntemi birlikte kullanabilir. Örneğin eski bir ATA denetleyicisi komutları port I/O ile gönderirken veri transferini DMA ile yapabilir. Bu durumda sıralama daha da önemlidir: komut portuna yazılan "başla" biti, DMA tanımlayıcısının bellekte görünür olduğundan emin olmadan gönderilmemelidir.

## Implementasyon notları

**Aygıtı keşfetmeden adres varsaymayın.** Sabit adresler yalnızca eski platformlarda ve belirli aygıtlarda geçerlidir. PCIe'de BAR, ACPI'de `_CRS`, aygıt ağacında `reg` kullanılmalıdır.

**Erişim genişliğini veri sayfasından doğrulayın.** 8/16/32/64 bit erişimlerden hangisinin desteklendiği aygıta göre değişir. "32 bit register" varsayımı, 16 bitlik bir alana 32 bit yazıp komşu register'ı bozabilir.

**`volatile`'ı sihirli çözüm sanmayın.** `volatile` yalnızca derleyicinin erişimi kaldırmasını engeller. Atomiklik, sıralama ve önbellek tutarlılığı sağlamaz.

**Okuma-geri-yazmayı aygıtın sözleşmesine göre yapın.** Write-one-to-clear bir bayrakta okuma-geri-yazma, istemediğiniz bayrakları da temizleyebilir veya yeniden tetikleyebilir.

**Bekleyen yazmaları unutmayın.** Aygıta "işi başlat" yazdıktan hemen sonra durum okumak, yazmanın henüz ulaşmadığı bir anda eski durumu görmenize yol açabilir. Aygıtın kendi "meşgul" bitini bekleme protokolü varsa ona uyun; yoksa aynı register'ı geri okuyarak posted yazmayı sürükleyin.

**MMIO eşlemesini doğru önbellek türüyle yapın.** Kontrol register'larını write-combining olarak eşlemek, sıralı yazma bekleyen aygıtlarda hataya yol açar. Framebuffer'ı uncached yapmak ise gereksiz performans kaybıdır.

**Aynı anda erişen CPU'ları düşünün.** MMIO erişimi tek başına atomik değildir. Aynı kontrol register'ına iki CPU yazıyorsa sürücü seviyesinde kilit gerekir. Aygıtta set/clear register'ları varsa read-modify-write yerine onlar tercih edilir.

**Sınırları ve taşmaları kontrol edin.** BAR boyutu ile erişim offset'i karşılaştırılmalıdır. Yanlış hesaplanan bir offset, komşu aygıtın register'ına yazabilir.

**Hata ayıklamada erişimleri görünür kılın.** MMIO okuma/yazmalarını geçici olarak loglamak, "yazdım ama aygıt görmedi" sorunlarını ayırmayı kolaylaştırır. Ancak loglamanın kendisi sıralamayı değiştirebilir; üretim kodunda bırakılmamalıdır.

## Mimariye göre farklar

| Mimari | Port I/O | MMIO | Önbellek/attribute |
| --- | --- | --- | --- |
| x86-64 | Var (`IN`/`OUT`, 64K alan) | Var | PCD/PWT, PAT |
| ARM64 | Yok | Var | MAIR_EL1, Device bellek türleri |
| RISC-V | Yok | Var | Platforma bağlı; Svpbmt sayfa bazında ayar |

ARM64'te `Device-nGnRnE` gibi bellek türleri, erişimlerin birleştirilmesini ve yeniden sıralanmasını kısıtlar. Hangi türün kullanılacağı aygıtın ve veri yolunun kurallarına bağlıdır; her MMIO bölgesi aynı türle eşlenmez. RISC-V'de Svpbmt yoksa bellek türleri platformun PMA (Physical Memory Attributes) yapılandırmasından gelir.

## Sık yapılan hatalar

- MMIO işaretçisini `volatile` yapmamak; derleyici erişimi siler.
- MMIO bölgesini önbelleğe alınabilir normal bellek olarak eşlemek.
- Aygıtın beklediği genişlikten farklı erişmek.
- `memcpy` ile MMIO alanına veri kopyalamak.
- DMA tanımlayıcısını yazmadan önce aygıta doorbell göndermek.
- Posted yazmanın tamamlandığını varsaymak.
- Aynı register'a kilitsiz read-modify-write yapmak.
- BAR boyutunu yanlış hesaplayıp komşu aralığa taşmak.
- Firmware'in atadığı BAR'ı yok sayıp sabit adres kullanmak.
- Port I/O'yu tüm mimarilerde var sanmak.

## İlgili notlar

- [MMIO ve port I/O](../01-mimari/mmio-ve-port-io.md)
- [Sürücü nedir, kernel içinde nerede durur](surucu-nedir.md)
- [Kesme tabanlı sürücüler ve interrupt handler tasarımı](kesme-tabanli-suruculer.md)
- [Seri port (UART)](uart.md)
- PCI ve PCIe aygıt keşfi: `pci.md`
- DMA: `dma.md`

## Kaynaklar

- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 2A ve 2B — `IN`, `OUT`, `INS`, `OUTS` komut tanımları
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 3A, Bölüm 11 — Memory Cache Control (PCD, PWT, PAT, bellek türleri)
- PCI-SIG, *PCI Local Bus Specification*, Revizyon 3.0 — Base Address Registers
- Arm® Architecture Reference Manual for A-profile architecture, Bölüm D8 — Memory Management (MAIR_EL1, Device memory)
- The RISC-V Instruction Set Manual, Volume II: Privileged Architecture — Svpbmt eklentisi
- Linux kernel dokümantasyonu, "Memory Barriers"; <https://docs.kernel.org/core-api/wrappers/memory-barriers.html> (erişim: 29 Ekim 2026)
- Linux kernel dokümantasyonu, "Bus-Independent Device Accesses"; <https://docs.kernel.org/driver-api/device-io.html> (erişim: 29 Ekim 2026)
- OSDev Wiki, "Memory Mapped I/O"; <https://wiki.osdev.org/Memory_Mapped_I/O> (erişim: 29 Ekim 2026)
