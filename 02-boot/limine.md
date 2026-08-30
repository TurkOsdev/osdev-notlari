# Limine Önyükleyicisi ve Önyükleme Protokolü

**Limine**, çeşitli mimari ve önyükleme protokollerini destekleyen açık kaynaklı bir önyükleyicidir. `IA-32`, `x86-64`, `arm64`, `riscv64` ve `loongarch64` mimarileriyle birlikte Linux, Limine, Multiboot (1 ve 2) ve Chainloading protokollerini destekler. Aynı zamanda Limine önyükleme protokolünün referans implementasyonudur.<sup>[1]</sup>

**Limine önyükleme protokolü** ise birden çok 64-bit mimariyi destekleyen önyükleme spesifikasyonudur. `x86-64`, `aarch64`, `riscv64` ve `loongarch64` mimarilerini destekler.<sup>[2]</sup> Spesifikasyonun tam haline [buradan](https://github.com/Limine-Bootloader/limine-protocol/blob/trunk/PROTOCOL.md) ulaşabilirsiniz, burada yer verilmeyen bütün özellikler orada mevcut.<sup>[2]</sup>

Limine, FAT12/16/32 ve ISO9660 dosya sistemlerini destekler. Doğru  yapılandırıldığında kernel çalıştırılabilir dosyanızı bu dosya sistemlerinden yükleyerek başlangıçta büyük kolaylık sağlar.<sup>[1]</sup>

## Ön koşullar

- [Ön gereksinimler](../00-giris/on-gereksinimler.md)
- [Geliştirme ortamı](../00-giris/gelistirme-ortami.md)
- [Bootloader yazmak mı, hazır bootloader mı?](./bootloader-secimi.md)
- Limine'ın sağladığı veri yapılarını anlayabilmek için C başlık dosyalarını (.h) okuyabilmelisiniz

## Kapsam

Bu not öncelikli olarak x86-64 üzerinde Limine ile çalışmaya nasıl başlanacağını anlatmak üzere yazılmıştır. Diğer mimariler için kaynaklar bölümündeki bağlantıları kullanabilirsiniz.

> Eğer diğer mimarilerde daha önce Limine kullandıysanız tecrübelerinizi bizimle paylaşabilirsiniz.

Önyükleme aşamasından sonra yapılması gereken şeylerden kısaca bahsedilecektir. Detaylar için konunun kendi notlarını okumanız önerilir.

## Problem

Önyükleme sürecinin birçok detayı vardır ve hedeflenen mimariye göre çok çeşitli farklılıklar içerebilir.  Bu farklılıklara birkaç örnek vermek gerekirse:

- x86-64 Legacy BIOS sistemlerde önyükleme diskin ilk sektöründeki (Master Boot Record) kodun belli bir adrese yüklenir ve 16-bit Gerçek Mod'da çalışmaya başlar.<sup>[3,4]</sup>
- x86-64 UEFI sistemlerde EFI sistem bölümündeki `.EFI` dosyaları çalıştırılır. Program 32-bit Korumalı Mod ya da 64-bit Long Mode'da çalışmaya başlar.<sup>[4]</sup>

Limine ve GNU GRUB gibi önyükleyiciler, kernel geliştiriclerini çok aşamalı önyükleme kodu yazmaktan ve diskten kernel'i bulma gibi problemlerden kurtarır. Ek olarak firmware'den ihtiyaç duyacağınız verileri tahmin edilebilir bir formatta kernel'inize verir.

`[02-boot](./)` bölümündeki diğer başlıklarda önyükleme süreciyle ilgili detaylı bilgilere ulaşabilirsiniz.

## Donanım seviyesinde

Limine, erken ortamda donanımdaki çeşitli farklılıkların bir kısmını ilk günden ele alma ihtiyacımızı ortadan kaldırır ve bellek haritası gibi önemli yapılara ulaşmak için erişilebilir bir soyutlama katmanı sunar.

## Kernel tarafında (x86-64)

### Bilinmesi gereken ön koşullar

Limine sadece higher-half kernel'leri destekler<sup>[2]</sup>, yani kernel kodunuz sanal adres alanının üst yarısında çalışır. Bu nedenle kernel'inizin üst yarıdaki yüksek bir adrese yükleneceğini varsayarak derlenmesi gerekir<sup>[5]</sup>. (Bunu linker scripti ile açıklayacağız)

Limine, kernel'inizi çalıştırmadan önce paging kurulumunu yapar ve kernel kodunuzun yaşadığı adres aralığı için sayfa tabloları oluşturur. [`limine_paging_mode_request`](https://github.com/Limine-Bootloader/limine-protocol/blob/trunk/PROTOCOL.md#paging-mode-feature) özelliği ile Limine'ın farklı bir paging modu kullanmasını isteyebilirsiniz.

Kernel'inizin ELF biçiminde derlenmesi önerilir<sup>[2]</sup>

Eğer kernel'inizde C/C++ dillerini kullanıyorsanız, `memcpy`, `memset`, `memcmp` ve `memmove` metotlarının mevcut olduğundan emin olun. [Burada basit bir implementasyona ulaşabilirsiniz.](https://github.com/Limine-Bootloader/limine-c-template-x86-64/blob/trunk/kernel/src/memory.c) Derleyiciler bu rutinlere doğrudan bir çağrı olmasa dahi bu rutinlere çağrılar oluşturabilir.

### Başlangıç ve temeller

> Buradaki kod parçalarının tam halini <https://github.com/Limine-Bootloader/limine-c-template-x86-64/tree/trunk> deposunda bulabilirsiniz. Bu şablonu klonlayarak Limine ile geliştirmeye hemen başlayabilirsiniz, ama aşağıdaki açıklamaları okumanız tavsiye edilir.

Kernel'iniz framebuffer, bellek haritası, RSDP, vb. gibi yapıları, [`limine.h`](https://github.com/Limine-Bootloader/limine-protocol/blob/trunk/include/limine.h) başlığında tanımlanan veri yapılarını kullanarak isteyebilir.

Örneğin aşağıdaki şekilde bellek haritasını tanımladığınız değişken içine alabilirsiniz:

```c
#include "limine.h" // bu başlığı doğrudan projenize kopyalayabilirsiniz

// derleyiciye bu değişkenin kullanıldığını, çıktıdan çıkarılmaması gerektiğini
// __attribute__((used)) ile söylüyoruz. section() attribute'u ise
// çıktıda hangi bölümde bulunacağını söylüyor.
//
// bunları birazdan linker scriptinde bir araya getireceğiz.
// 
// derleyicinin bu değişkenlerle bilmediğiniz işler yapmasını
// önlemek için `volatile` eklemeyi unutmayın

// önyükleyiciye istediğimiz revizyonu belirtiyoruz
__attribute__((used, section(".limine_requests")))
static volatile uint64_t limine_base_revision[] = LIMINE_BASE_REVISION(6);

__attribute__((used, section(".limine_requests")))
static volatile struct limine_framebuffer_request framebuffer_request = {
    .id = LIMINE_FRAMEBUFFER_REQUEST_ID,
    .revision = 0
};

__attribute__((used, section(".limine_requests_start")))
static volatile uint64_t limine_requests_start_marker[] = LIMINE_REQUESTS_START_MARKER;

__attribute__((used, section(".limine_requests_end")))
static volatile uint64_t limine_requests_end_marker[] = LIMINE_REQUESTS_END_MARKER;
```

Limine, kernel'inizi yüklerken çeşitli sabit sayıları arayarak isteklerinizi bulur ve beklenen adrese yanıtı yerleştirir. Bu yanıtların `NULL` olup olmadığını mutlaka kontrol etmelisiniz. Örneğin, UEFI firmware olmayan bir makinede `limine_efi_system_table_request` herhangi bir yanıt döndürmez<sup>[2]</sup>

Çalıştırılabilir dosyanızda tanımladığınız isteklerin `LIMINE_REQUESTS_START_MARKER` sayıları ile `LIMINE_REQUESTS_END_MARKER` arasında olması gereklidir. Ancak bunların C kodunuzda nerede bulunduklarının bir önemi yoktur, o yüzden yukarıdaki kod parçasında olduğu gibi sonda bırakabilirsiniz. Sıralamayı linker scripti ile zorlayacağız.

Devam etmeden önce giriş noktamızı da oluşturalım:

```c

// uint*_t tipleri için
#include <stdint.h>

// NULL için
#include <stddef.h>
#include <stdbool.h>

// "halt and catch fire" fonksiyonu bir
// interrupt gelinceye kadar yürütmeyi
// durdurur. döngü kullanarak her interrupt'tan
// sonra da beklemeye devam etmesi sağlanır.
//
// buraya interrupt'ları ayarlamadan girmek
// panic ile eşdeğerdir, makine anlamlı bir
// şey hiçbir zaman yapmaz.
static void hcf(void) {
  // bu talimat mimariler arası değişkenlik gösterebilir.
  // #if defined(__x86_64__) gibi direktifler kullanarak
  // farklı mimariler için de implemente edebilirsiniz.
  for (;;) asm("hlt");
}

void kmain(void) {
  // bootloader isteklerimizi destekliyor mu?
  if (LIMINE_BASE_REVISION_SUPPORTED(limine_base_revision) == false) {
        // devam etmek anlamsız, önyükleyicinin istediğimiz verileri
        // sağladığının garantisi yok.
        hcf();
  }

  // yanıtlar struct'ın .response alanına yerleştirilir. NULL olmadığından
  // emin oluyoruz
  if (framebuffer_request.response == NULL 
      || framebuffer_request.response->framebuffer_count < 1) {
    hcf();
  }

  // listedeki ilk framebufferı alıyoruz
  struct limine_framebuffer *framebuffer = framebuffer_request.response->framebuffers[0];

  // ekrana bir şeyler çizelim
  // (çalıştırdığınızda bir renk geçişi görmelisiniz)
  volatile uint32_t *fb_ptr = framebuffer->address;
  for (size_t y = 0; y < framebuffer->height; y++) {
      for (size_t x = 0; x < framebuffer->width; x++) {
          uint32_t nX = x * 255 / framebuffer->width;
          uint32_t nY = y * 255 / framebuffer->height;
          fb_ptr[y * (framebuffer->pitch / 4) + x] = (nY << 8) | nX;
      }
  }

  // işimiz bitti
  hcf();
}
```

### Linker script

Script'in tam haline [buradan](https://github.com/Limine-Bootloader/limine-c-template-x86-64/blob/trunk/kernel/linker-scripts/x86_64.lds) ulaşabilirsiniz. Burada sadece önemli/anlamanız gereken kısımları (yukarıdan aşağı) açıklayacağız.

Linker'dan x86_64 ELF formatında bir çıktı istiyoruz.

```ld
OUTPUT_FORMAT(elf64-x86-64)
```

Giriş noktamıza işaret eden sembol. Eğer Rust ya da C++ kullanacaksanız mutlaka `extern "C"` olarak tanımlayın, aksi takdirde derleyici ismi bozacaktır. Rust kullandıysanız `#[unsafe(no_mangle)]` makrosunu da eklediğinizden emin olun.

```ld
ENTRY(kmain)
```

Şimdi yürütülebilir dosyada neyin nerede olacağını belirliyoruz. `.` olan yer bulunduğunuz konuma işaret eder. Düz bir yolda anlık olarak bastığınız nokta gibi düşünebilirsiniz.

```ld
SECTIONS
{
    /* adres alanının en üst 2GB'lık alanına yerleşmek istiyoruz. */
    /* limine kernel'i buraya yerleştireceği için bütün sembollerin */
    /* bu adrese göre hesaplanmış olmasını sağlamalıyız. */
    . = 0xffffffff80000000;

    /* __attribute((section())) yazdığımız yeri hatırladınız mı? */
    /* işte burada onları sıraya koyuyoruz. önce start, sonra istekler, */
    /* en sonda da bitiş işareti. */
    .limine_requests : {
        KEEP(*(.limine_requests_start))
        KEEP(*(.limine_requests))
        KEEP(*(.limine_requests_end))
    } :limine_requests

   
   /* .text alanını sonraki page'e hizalıyoruz. */
    . = ALIGN(CONSTANT(MAXPAGESIZE));

    /* yürütülebilir kodlar bu alanda bulunur */
    .text : {
        *(.text .text.*)
    } :text

    /* yine sonraki page'e hizalıyoruz. bu sefer .rodata */
    /* (read-only data) için */
    . = ALIGN(CONSTANT(MAXPAGESIZE));

  .rodata : {
        *(.rodata .rodata.*)
  } :rodata

  /* eğer bir tanımlayıcı verilmişse onu buraya kaydediyoruz */
  /* buraya şu an takılmanıza gerek yok. */
  .note.gnu.build-id : {
        *(.note.gnu.build-id)
  } :rodata

  /* yine sayfa atlıyoruz, bu sefer .data için */
  . = ALIGN(CONSTANT(MAXPAGESIZE));

  .data : {
      *(.data .data.*)
  } :data

  /* bu alanda henüz bir değer atamadığınız (0 ya da NULL olan) */
  /* statik değişkenleriniz bulunur. bu alanlar için program yükleyicisi */
  /* bellekten alan ayırır, 0'lar yürütülebilir çıktınıza dahil edilmez. */
  /* büyük bir diziniz varsa onlar çıktıda yer kaplamaz, ancak yüklendikten */
  /* sonra bellekte yer kaplar. */
  .bss : {
      *(.bss .bss.*)
      *(COMMON)
  } :data

  /* dışlamak istediğimiz alanlar. .note ve .eh_frame bazı makinelerde */
  /* problem yaratabiliyor, o yüzden atıyoruz. */
  /DISCARD/ : {
      *(.eh_frame*)
      *(.note .note.*)
  }

}
```

### `limine.conf`<sup>[7][dokümantasyon](https://github.com/Limine-Bootloader/Limine/blob/v12.x/CONFIG.md)</sup>

`timeout` değişkeni otomatik olarak kernel'i yüklemeden önce kaç saniye bekleneceğini belirler. Test ortamınızda başka bir işletim sistemi yoksa/boot menü kullanmanız gerekmiyorsa bunu düşürmeniz önerilir.

`/Limine Template` menüde gösterilecek bir girdi tanımlar.

- `protocol` ile "limine" önyükleme protokolünü kullanmak istediğimizi belirtiyoruz. Bu ayarı kullanarak Linux ve Multiboot gibi protokolleri destekleyen işletim sistemlerini de yükleyebilirsiniz, ancak biz sadece limine protokolünü kullanacağız.
- `path` ile kernelimizin bulunduğu yeri belirtiyoruz (birazdan gerçekten oraya koyacağız)

```conf
timeout: 3

/Limine Template
    protocol: limine
    path: boot():/boot/kernel
```

### Derleme ve çalıştırma

`limine-c-template` içindeki GNUmakefile ile projenizi derleyip çalıştırabilirsiniz. Sağlanan Makefile otomatik olarak Limine'ın derlenmiş bir dağıtımını indirir ve ISO görüntüsü oluşturur.

## Implementasyon notları

Kernel'inizde birden çok önyükleme protokolünü destekleyebilirsiniz, ancak her protokolün sizi farklı bir durumda bırakabileceğinin farkında olmalısınız.

Limine'a ait veri tiplerini bütün kod tabanınıza sızdırmamanız fayda sağlayacaktır. Örneğin bir grafik sürücüsü yazarken kendi `Framebuffer` tipinizi kullanmak `struct limine_framebuffer` kullanmaya göre daha fazla özgürlük tanıyacaktır. Eğer ileride farklı bir önyükleme kullanmak isterseniz kernel'inizin diğer bölümleri bu değişiklikten en az seviyede etkilenmeli.

Limine HHDM ile sayfa tablolarınızı ayarlar, ancak hangi bellek alanlarının haritalandırıldığı protokolün sürümleri arasında değişiklik gösterebildiği için kendi sayfa tablolarınızı oluşturmanız önerilir.

## Sık yapılan hatalar

- Önyükleme katmanınına ait şeylerin kernel mantığınıza karışması. Almanız gereken verileri kendi tanımladığınız bir arayüz üzerinden alın. Limine protokolünün sürümleri arasında birçok değişiklik var.
- Limine'ın sayfa tablolarına güvenmek. Limine'ın haritalandırdığı bellek alanları protokolün sürümleri arasında farklılık gösterir: Örneğin base revision 0 bütün bellek haritası alanları için sayfa tablosu oluşturuyordu. Revision 1 "Reserved" ve "Bad memory" alanlarını tablodan çıkardı. Revision 3 sadece kullanılabilir, framebuffer, kernel ve yeniden kazanılabilir alanlar için sayfa tabloları oluşturuyordu. Bu tip farklılıklar çalışırken sizi şüpheye düşürür, bu yüzden kullanacağınız alanları kendiniz haritalandırın ve sayfa tablolarınızı güvendiğiniz bir alanda saklayın.<sup>[2:Base Revision Changes Summary]
- Limine'ın veri yapılarına doğrudan bağımlı olmak. Bu yapılar protokol güncellendikçe değişebileceğinden kendi mantığınızı yazarken kullanmayın. Kısaca önyükleme işleriyle ilgilenen dosya haricinde `#include <limine.h>` bulundurmamaya çalışın.
- Response'lara `NULL` kontrolü uygulamamak. Her isteğe her platformda yanıt geleceğinin garantisi yoktur.

## İlgili notlar

- [Bootloader yazmak mı, hazır bootloader mı?](./bootloader-secimi.md)
- [Boot sürecine genel bakış](./boot-sureci.md)
- [BIOS ve legacy boot](./bios.md)
- [MBR ve boot sector](./mbr.md)

## Kaynaklar

- [1] [Limine-Bootloader/Limine:README.md](https://github.com/Limine-Bootloader/Limine/blob/v12.x/README.md)
- [2] [Limine Boot Protocol spesifikasyonu](https://github.com/Limine-Bootloader/limine-protocol/blob/trunk/PROTOCOL.md)
- [3] <https://en.wikipedia.org/wiki/Master_boot_record> / Erişim: 28 Ağustos 2026 00:18 (UTC+3)
- [4] <https://osdev.wiki/wiki/Boot_Sequence> / Erişim: 28 Ağustos 2026 12:21 (UTC+3)
- [5] <https://wiki.osdev.org/Higher_Half_Kernel> / Erişim: 28 Ağustos 2026 00:45 (UTC+3)
- [6] <https://github.com/Limine-Bootloader/limine-c-template-x86-64>
- [7] <https://github.com/Limine-Bootloader/Limine/blob/v12.x/CONFIG.md>
