# Limine Önyükleyicisi ve Önyükleme Protokolü

**Limine**, işletim sistemlerini ve freestanding programları başlatmak için kullanılan modern, taşınabilir ve çoklu protokol desteğine sahip açık kaynaklı bir **bootloader ve boot manager**'dır.

Aynı zamanda **Limine Boot Protocol**'ün referans implementasyonudur.

Limine'ın temel amacı kernel geliştiricisini firmware, disk erişimi, boot aşamaları ve ilk bellek düzeni gibi karmaşık ayrıntıların önemli bir bölümünden kurtarıp kernel'i tanımlı ve kullanılabilir bir ortamda çalıştırmaktır.

Bir kernel geliştirirken Limine kullandığınızda örneğin:

* Kernel dosyasını diskten bulmak,
* ELF dosyasını belleğe yüklemek,
* İlk page table'ları oluşturmak,
* Framebuffer bilgilerini almak,
* Firmware'den bellek haritasını edinmek,
* ACPI RSDP gibi sistem tablolarını bulmak,
* Kernel modüllerini yüklemek,

gibi birçok erken önyükleme işlemini kendiniz gerçekleştirmek zorunda kalmazsınız.

---

## Limine Bootloader ve Limine Boot Protocol aynı şey değildir

Bu iki kavramın ayrımını yapmak önemlidir.

### Limine Bootloader

Limine'ın kendisi fiziksel veya sanal makine üzerinde çalışan önyükleyicidir.

Desteklediği mimariler:

* IA-32
* x86-64
* aarch64
* riscv64
* loongarch64

Limine yalnızca kendi protokolüyle kernel yüklemek zorunda değildir.

Desteklediği önyükleme protokolleri arasında:

* Limine Boot Protocol
* Linux boot protocol
* Multiboot 1
* Multiboot 2
* Chainloading

bulunur.

Bu nedenle Limine yalnızca hobby kernel geliştirmek için kullanılan bir araç değil, farklı işletim sistemlerini başlatabilen genel amaçlı bir boot manager olarak da düşünülebilir.

### Limine Boot Protocol

**Limine Boot Protocol**, bootloader ile kernel arasında nasıl bilgi aktarılacağını belirleyen protokoldür.

Protokol:

* x86-64
* aarch64
* riscv64
* loongarch64

mimarilerini destekler.

IA-32, Limine bootloader tarafından desteklenmesine rağmen Limine Boot Protocol yalnızca **64-bit little-endian** sistemleri hedefler.

Kernel'in hangi çalıştırılabilir formatta olması gerektiği protokol tarafından zorunlu tutulmaz fakat **ELF kullanılması tavsiye edilir.**

---

# Neden Limine?

Bir kernel'in çalışmaya başlayabilmesi için CPU'nun reset durumundan kernel giriş noktasına ulaşmasına kadar birçok işlem gerçekleştirilmesi gerekir.

Bu süreç platforma göre ciddi şekilde değişebilir.

Örneğin x86 Legacy BIOS üzerinde:

1. BIOS diskin boot sector'ünü belleğe yükler.
2. CPU 16-bit Real Mode içerisinde kod çalıştırmaya başlar.
3. Kernel geliştiricisinin A20, GDT, Protected Mode, Long Mode ve paging gibi aşamalarla ilgilenmesi gerekebilir.
4. Kernel dosyasının diskten bulunması ve yüklenmesi gerekir.

UEFI üzerinde ise tamamen farklı bir firmware ortamı ve API bulunur.

ARM64 veya RISC-V tarafında başlangıç durumu yine farklıdır.

Limine bu farklılıkların önemli bir bölümünü bootloader katmanında ele alır.

Genel akış kabaca şöyledir:

```text
Firmware
   │
   ▼
Limine
   │
   ├── Kernel dosyasını bul
   ├── Kernel'i belleğe yükle
   ├── Memory map al
   ├── Page table'ları hazırla
   ├── Framebuffer / ACPI / DTB vb. bilgileri hazırla
   │
   ▼
Kernel entry point
   │
   ▼
Kernel
```

Böylece kernel geliştiricisi doğrudan kernel tarafındaki problemlere odaklanabilir.

---

# Firmware desteği

Limine farklı firmware ortamlarında çalışabilir.

## x86

x86 sistemlerde:

* Legacy BIOS
* UEFI

ile kullanılabilir.

Bu sayede aynı işletim sistemi modern UEFI makinelerin yanında eski BIOS makinelerde de Limine üzerinden başlatılabilir.

## Diğer mimariler

aarch64, riscv64 ve loongarch64 tarafında Limine UEFI ortamında kullanılabilir.

Kernel'in Limine Boot Protocol tarafından teslim edildiği CPU durumu ise her mimaride aynı değildir.

Bu nedenle bootloader kullanmak mimariye özel kernel kodunu tamamen ortadan kaldırmaz.

---

# Disk ve dosya sistemi desteği

Limine aşağıdaki partition yapılarını destekler:

* GPT
* MBR
* Partition tablosu bulunmayan medya

Desteklenen dosya sistemleri:

* FAT12
* FAT16
* FAT32
* ISO9660

Kernel'in mutlaka root filesystem'inizde bulunması gerekmez.

Örneğin root filesystem'iniz ext2 olsa bile kernel ve Limine dosyalarını FAT32 biçimindeki ayrı bir EFI System Partition içerisinde bulundurabilirsiniz.

Burada önemli olan bir ayrım vardır:

**Bootloader'ın kernel'i okuyabildiği dosya sistemi ile kernel'in daha sonra mount edeceği root filesystem birbirinden bağımsızdır.**

---

# Limine kernel'e ne sağlar?

Limine Boot Protocol bir **request/response** sistemi kullanır.

Kernel, çalıştırılabilir dosyasının içerisinde belirli Limine request yapılarını bulundurur.

Örneğin kernel:

* framebuffer
* memory map
* HHDM adresi
* firmware tipi
* ACPI RSDP
* SMBIOS
* EFI System Table
* kernel dosyası
* ek modüller
* SMP bilgileri
* paging modu
* Device Tree Blob
* bootloader bilgileri

gibi verileri isteyebilir.

Bootloader kernel'i yüklerken bu request yapılarını bulur ve desteklediği isteklerin `response` alanlarını doldurur.

Kabaca:

```text
Kernel ELF
 ├── Kod
 ├── Data
 │
 └── Limine Requests
       │
       ├── "Memory map istiyorum"
       ├── "Framebuffer istiyorum"
       └── "HHDM adresini istiyorum"
              │
              ▼
           Limine
              │
              ▼
       Response yapılarını doldurur
              │
              ▼
           Kernel başlar
```

Bu tasarım sayesinde kernel giriş fonksiyonuna devasa bir boot information struct geçirilmesi gerekmez.

Kernel yalnızca gerçekten ihtiyacı olan özellikleri talep eder.

---

# Request sistemi

Örneğin framebuffer istemek için:

```c
#include <limine.h>

__attribute__((used, section(".limine_requests")))
static volatile struct limine_framebuffer_request framebuffer_request = {
    .id = LIMINE_FRAMEBUFFER_REQUEST_ID,
    .revision = 0
};
```

Limine kernel'i yüklediğinde isteği destekliyorsa:

```c
framebuffer_request.response
```

alanına bir cevap yerleştirir.

Ancak her request'in mutlaka cevaplanacağının garantisi yoktur.

Bu nedenle:

```c
if (framebuffer_request.response == NULL) {
    // framebuffer mevcut değil
}
```

gibi kontroller yapılmalıdır.

Örneğin UEFI ile ilgili bir request BIOS üzerinden başlatılan bir sistemde anlamlı bir cevap vermeyebilir.

---

# Base Revision

Limine Boot Protocol zaman içerisinde geliştirildiği için protokolün farklı **base revision** sürümleri bulunmaktadır.

Kernel hangi temel protokol davranışını beklediğini bootloader'a bildirir.

Örneğin:

```c
__attribute__((used, section(".limine_requests")))
static volatile uint64_t limine_base_revision[] =
    LIMINE_BASE_REVISION(6);
```

Kernel çalışmaya başladıktan sonra bootloader'ın istediği revision'ı destekleyip desteklemediği kontrol edilmelidir.

```c
if (!LIMINE_BASE_REVISION_SUPPORTED(limine_base_revision)) {
    // devam edilmemeli
}
```

Base revision yalnızca bir sürüm numarası değildir.

Revision değiştikçe örneğin:

* başlangıçtaki CPU durumu,
* hangi bellek bölgelerinin map edildiği,
* bazı firmware adreslerinin nasıl döndürüldüğü,

gibi protokol davranışları değişebilir.

Bu nedenle kernel'in yalnızca tesadüfen belirli bir Limine sürümünde çalışan varsayımlara güvenmemesi gerekir.

---

# Requests alanı

Limine request'leri kernel executable içerisinde belirli bir bölgede tutulabilir.

Başlangıç:

```c
__attribute__((used, section(".limine_requests_start")))
static volatile uint64_t limine_requests_start_marker[] =
    LIMINE_REQUESTS_START_MARKER;
```

Request'ler:

```c
__attribute__((used, section(".limine_requests")))
static volatile struct limine_memory_map_request memmap_request = {
    .id = LIMINE_MEMMAP_REQUEST_ID,
    .revision = 0
};
```

Bitiş:

```c
__attribute__((used, section(".limine_requests_end")))
static volatile uint64_t limine_requests_end_marker[] =
    LIMINE_REQUESTS_END_MARKER;
```

Linker script içerisinde bu alanların sırası korunmalıdır.

```ld
.limine_requests : {
    KEEP(*(.limine_requests_start))
    KEEP(*(.limine_requests))
    KEEP(*(.limine_requests_end))
}
```

Bu mekanizma mimariye özel değildir; Limine Boot Protocol'ün genel request modelinin parçasıdır.

---

# Memory Map

Kernel'in ilk ihtiyaç duyacağı yapılardan biri fiziksel bellek haritasıdır.

Limine'dan memory map isteyebilirsiniz:

```c
__attribute__((used, section(".limine_requests")))
static volatile struct limine_memmap_request memmap_request = {
    .id = LIMINE_MEMMAP_REQUEST_ID,
    .revision = 0
};
```

Kernel başladıktan sonra response içerisindeki entry'ler incelenebilir.

Bellek haritasında alanlar örneğin:

* kullanılabilir RAM,
* kernel'in bulunduğu alanlar,
* bootloader tarafından kullanılan alanlar,
* framebuffer,
* ACPI bölgeleri,

gibi farklı tiplerde bulunabilir.

Physical Memory Manager yazarken kullanılabilir RAM bölgelerini tespit etmek için bu harita kullanılabilir.

---

# HHDM

Limine'ın sağladığı önemli özelliklerden biri **Higher Half Direct Map (HHDM)**'dir.

HHDM fiziksel belleğin kernel'in sanal adres alanındaki belirli bir offset üzerinden erişilebilir hale getirilmesini sağlar.

Basit düşünürsek:

```text
physical address
       +
HHDM offset
       =
virtual address
```

Kernel HHDM offset'ini request aracılığıyla öğrenebilir.

Önemli olarak bu offset'in sabit olduğu varsayılmamalıdır.

Kernel her boot sırasında Limine tarafından verilen değeri kullanmalıdır.

---

# Paging

Limine kernel çalışmaya başlamadan önce kullanılabilir page table'lar hazırlar.

Ancak bu page table'ları kernel'in kalıcı paging altyapısı olarak görmek iyi bir tasarım değildir.

Bootloader tarafından oluşturulan tablolar kernel'in **başlangıç ortamıdır**.

Kernel kendi memory manager'ı hazır olduğunda kendi:

* page table'larını,
* memory mapping politikasını,
* permission'larını,
* address-space düzenini

oluşturmalıdır.

Limine'ın paging-mode request'i ile mimariye göre bazı paging özellikleri de talep edilebilir.

Bunlar mimariye göre farklıdır.

Örneğin:

```text
x86-64
 ├── 4-level paging
 └── 5-level paging

aarch64
 ├── 4-level
 └── 5-level

riscv64
 ├── Sv39
 ├── Sv48
 └── Sv57
```

Bu, Limine'ın ortak bir protokol sağlamasına rağmen donanım mimarisini kernel'den gizlemediğinin güzel bir örneğidir.

---

# Kernel'in başlangıç durumu

Limine bütün mimarileri tamamen aynı CPU durumunda bırakmaz.

Her mimarinin kendi ABI'si ve mimari özellikleri vardır.

Limine Boot Protocol tarafından kullanılan ABI'ler genel olarak:

| Mimari      | ABI              |
| ----------- | ---------------- |
| x86-64      | System V ABI     |
| aarch64     | AAPCS64          |
| riscv64     | LP64 soft-float  |
| loongarch64 | LP64S soft-float |

Kernel entry point'iniz ilgili mimarinin ABI kurallarına uygun olmalıdır.

Ayrıca interrupt, exception, descriptor table veya trap vector gibi mimariye özel yapıları kernel'in kendisi kurmalıdır.

Örneğin x86-64 kernel:

```text
GDT
IDT
APIC
CR3
```

gibi yapılarla ilgilenirken RISC-V kernel:

```text
stvec
satp
S-mode
PLIC / AIA
```

gibi tamamen farklı yapılarla çalışacaktır.

**Limine sizi boot sürecinden kurtarır; kernel mimarisinden kurtarmaz.**

---

# Framebuffer

Limine firmware tarafından sağlanan framebuffer'ı kernel'e verebilir.

Bu sayede daha GPU sürücüsü yazmadan ekrana piksel çizmek mümkündür.

```c
struct limine_framebuffer *fb =
    framebuffer_request.response->framebuffers[0];
```

Framebuffer yapısı içerisinde:

* adres,
* genişlik,
* yükseklik,
* pitch,
* bits per pixel,
* renk maskeleri

gibi bilgiler bulunur.

Framebuffer bir GPU hızlandırma API'si değildir.

Kernel'e yalnızca belleğe yazılarak görüntü oluşturulabilen bir framebuffer sağlar.

---

# SMP

Limine, desteklenen platformlarda kernel'in diğer işlemci çekirdeklerini başlatmasına yardımcı olacak SMP bilgilerini sağlayabilir.

Bu özellik sayesinde kernel:

* CPU sayısını,
* CPU kimliklerini,
* bootstrap processor bilgisini

öğrenebilir ve diğer işlemciler için başlangıç adresleri belirleyebilir.

Ancak scheduler, per-CPU veri yapıları, locking ve çekirdekler arası senkronizasyon yine kernel'in sorumluluğundadır.

---

# ACPI ve Device Tree

Donanım keşfi her platformda aynı şekilde yapılmaz.

x86 sistemlerde çoğunlukla **ACPI** kullanılırken ARM ve RISC-V sistemlerinde **Device Tree** oldukça yaygındır.

Limine:

* RSDP
* SMBIOS
* Device Tree Blob
* EFI System Table

gibi firmware yapılarını kernel'e aktarabilir.

Böylece kernel'in firmware belleğini rastgele tarayarak bu yapıların adreslerini bulması gerekmez.

---

# Modüller

Kernel dışındaki dosyalar da Limine tarafından belleğe yüklenebilir.

Bunlara Limine terminolojisinde **module** denir.

Örneğin:

```text
kernel
initramfs
font.psf
config
```

gibi dosyalar boot sırasında belleğe yüklenip kernel'e aktarılabilir.

Bu özellikle initramfs kullanan işletim sistemlerinde oldukça kullanışlıdır.

---

# `limine.conf`

Limine'ın boot menüsü ve boot entry'leri `limine.conf` ile yapılandırılır.

Basit bir örnek:

```conf
timeout: 3

/Tunix
    protocol: limine
    path: boot():/boot/kernel
```

Burada:

* `/Tunix` boot menüsünde gösterilecek girdidir.
* `protocol` kullanılacak boot protokolünü belirler.
* `path` kernel executable'ın konumunu belirtir.

Limine config sistemi yalnızca kernel yolu belirlemekten ibaret değildir.

Boot entry'leri, modüller ve çeşitli protokol seçenekleri burada yapılandırılabilir.

Ayrıca Limine bazı built-in macro'lar sağlar:

```text
${ARCH}
${FW_TYPE}
${LOADER_ARCH}
```

Örneğin `${ARCH}` çalışan makinenin mimarisine göre:

```text
x86-64
ia-32
aarch64
riscv64
loongarch64
```

değerlerinden birine genişletilebilir.

Bu özellik multi-architecture boot medyaları hazırlarken kullanışlıdır.

---

# Mimariler arası kernel tasarımı

Limine birden fazla mimariyi desteklediği için kernel kodunuzu baştan sadece x86-64'e bağımlı yazmamak faydalıdır.

Örneğin:

```text
kernel/
├── arch/
│   ├── x86_64/
│   ├── aarch64/
│   ├── riscv64/
│   └── loongarch64/
│
├── boot/
│   └── limine.c
│
├── mm/
├── sched/
├── fs/
└── kernel/
```

gibi bir yapı kullanılabilir.

Limine'a ait yapıları da mümkün olduğunca `boot/` katmanında tutmak mantıklıdır.

Örneğin kernel'in grafik kodunda:

```c
struct limine_framebuffer
```

kullanmak yerine kendi yapınızı:

```c
struct framebuffer {
    void *address;
    uint64_t width;
    uint64_t height;
    uint64_t pitch;
};
```

tanımlayabilirsiniz.

Limine katmanı gelen veriyi kernel'in kendi veri tiplerine çevirir:

```text
Limine
  │
  ▼
boot/limine.c
  │
  ▼
Kernel internal API
  │
  ├── memory manager
  ├── graphics
  ├── ACPI
  └── scheduler
```

Böylece ileride Limine yerine başka bir boot protocol eklemek istediğinizde kernel'in tamamını değiştirmek zorunda kalmazsınız.

---

# Mimariye özel bölümler

Buraya kadar anlatılan özelliklerin büyük bölümü Limine Boot Protocol'ün ortak çalışma modelidir.

Ancak aşağıdaki konular mimariye özel olarak ele alınmalıdır:

### x86-64

* Higher-half linker script
* Long Mode başlangıç durumu
* GDT
* IDT
* APIC
* 4/5-level paging
* `hlt`

### aarch64

* Exception Levels
* `VBAR_EL1`
* AArch64 page tables
* GIC
* Device Tree / ACPI
* `wfi`

### riscv64

* Supervisor Mode
* `stvec`
* `satp`
* Sv39 / Sv48 / Sv57
* SBI
* PLIC / AIA
* `wfi`

### loongarch64

* CSR başlangıç durumu
* paging
* interrupt controller
* mimariye özel exception altyapısı

Bu nedenle örneğin linker script veya `halt()` implementasyonu anlatılırken bunun **Limine'ın kendisine değil hedef mimariye ait olduğu açıkça belirtilmelidir.**

---

# Limine'dan sonra ne olur?

Limine kernel'inize kontrolü verdiğinde gerçek kernel geliştirme süreci yeni başlamıştır.

Tipik olarak kernel bundan sonra:

1. Limine response'larını kendi veri yapılarına kopyalar.
2. Fiziksel bellek yöneticisini başlatır.
3. Kendi page table'larını oluşturur.
4. Exception ve interrupt altyapısını kurar.
5. Timer'ı başlatır.
6. SMP desteğini hazırlar.
7. Device discovery yapar.
8. Sürücüleri başlatır.
9. VFS ve dosya sistemlerini hazırlar.
10. Scheduler'ı başlatır.
11. İlk userspace sürecini çalıştırır.

Bu noktadan sonra Limine'ın görevi fiilen bitmiştir.

Kernel, bootloader'dan bağımsız şekilde sistemi yönetmeye başlar.

---

# Sık yapılan hatalar

### Limine yapılarını kernel'in her yerine yaymak

Kernel'in birçok yerinde:

```c
#include <limine.h>
```

bulunması ileride boot katmanını değiştirmeyi zorlaştırır.

Limine ile iletişimi mümkün olduğunca tek bir boot katmanında tutun.

### Response'ların her zaman geleceğini varsaymak

Bir request'in tanımlanmış olması onun mutlaka cevaplanacağı anlamına gelmez.

Her gerekli response için `NULL` kontrolü yapılmalıdır.

### Bootloader page table'larına kalıcı olarak güvenmek

Limine tarafından hazırlanan paging ortamını kernel'in başlangıç ortamı olarak değerlendirin.

Memory manager hazır olduğunda kontrolü kendi page table'larınıza geçirin.

### HHDM adresini sabit varsaymak

HHDM offset bootloader tarafından belirlenir ve kernel tarafından request üzerinden öğrenilmelidir.

### Bootloader ile firmware'i aynı şey sanmak

Limine firmware değildir.

Örneğin:

```text
UEFI
 ↓
Limine
 ↓
Kernel
```

üç ayrı yazılım katmanıdır.

### Limine'ın donanım soyutlama katmanı olduğunu düşünmek

Limine size başlangıç bilgilerini sağlar fakat kernel'in yerine:

* scheduler,
* filesystem,
* driver,
* interrupt handler,
* network stack,
* memory manager

oluşturmaz.

Limine'ın işi sistemi **boot etmek ve kernel'e temiz bir başlangıç ortamı sağlamaktır.**

---

# Sonuç

Limine yalnızca kernel dosyasını belleğe atan küçük bir bootloader değildir.

Farklı firmware ve mimariler üzerinde çalışan, birden fazla boot protokolünü destekleyen ve özellikle Limine Boot Protocol aracılığıyla kernel geliştiricilerine güçlü bir başlangıç ortamı sağlayan bir boot altyapısıdır.

En önemli avantajlarından biri kernel'in firmware ve erken boot sürecindeki birçok platform bağımlı ayrıntıyla doğrudan ilgilenme ihtiyacını azaltmasıdır.

Ancak Limine kernel ile donanım arasındaki mimari farklılıkları ortadan kaldırmaz.

Bu nedenle iyi tasarlanmış bir kernel:

```text
Firmware
    ↓
Limine
    ↓
Boot abstraction
    ↓
Architecture layer
    ↓
Kernel core
```

şeklinde katmanlara ayrılabilir.

Limine burada sistemin geri kalanını oluşturmaz; kernel'in güvenilir ve tanımlanmış bir noktadan geliştirmeye başlayabilmesini sağlar.

## İlgili notlar

* [Bootloader yazmak mı, hazır bootloader mı?](./bootloader-secimi.md)
* [Boot sürecine genel bakış](./boot-sureci.md)
* [BIOS ve legacy boot](./bios.md)
* [MBR ve boot sector](./mbr.md)

## Kaynaklar

* [Limine Bootloader](https://github.com/Limine-Bootloader/Limine)
* [Limine Boot Protocol](https://github.com/Limine-Bootloader/limine-protocol/blob/trunk/PROTOCOL.md)
* [Limine Configuration Documentation](https://github.com/Limine-Bootloader/Limine/blob/v12.x/CONFIG.md)
* [Limine C Template](https://github.com/Limine-Bootloader/limine-c-template)
