# Build sistemi: Make ve alternatifleri

Kernel'i derlemek tek bir derleyici komutundan ibaret değildir. C ve assembly kaynakları nesne dosyalarına dönüştürülür, bu dosyalar linker script'e göre bağlanır, ortaya çıkan kernel bir boot imajına yerleştirilir ve sonuç emülatörde çalıştırılır. Build sistemi bu adımların hangi sırada ve ne zaman çalışacağını tarif eder.

Bu tarif yalnızca kolaylık sağlamaz. Bir header değiştiğinde ondan etkilenen dosyaların yeniden derlenmesini, birbirinden bağımsız işlerin paralel yürütülmesini ve herkesin aynı komutlarla aynı çıktıları üretmesini sağlar. Doğru kurulmuş bir build sistemi, eski bir nesne dosyasının yeni kernel'e karışması gibi teşhisi zor hataları daha ortaya çıkmadan engeller.

## Ön koşullar

- [Geliştirme ortamı kurulumu](gelistirme-ortami.md)
- [Cross-compiler hazırlama](cross-compiler.md)

## Kapsam

Bu not, x86-64 hedefleyen küçük bir kernel projesini ve GNU Make'i esas alır. Make kuralları mimariden bağımsızdır; örnekteki derleyici adları ve bayraklar ise x86-64'e özgüdür. Bootloader'a göre değişen imaj üretim adımı ayrı tutulmuştur.

## Build sisteminin çözdüğü problem

İlk denemelerde komutları elle çalıştırmak yeterli görünebilir:

```sh
x86_64-elf-gcc -c src/kernel.c -o kernel.o
x86_64-elf-gcc -c src/serial.c -o serial.o
x86_64-elf-gcc -T linker.ld -o kernel.elf kernel.o serial.o
```

Proje büyüdüğünde bu yöntem üç nedenle sorun çıkarır:

- Değişmeyen her dosya da yeniden derlenir veya hangi dosyanın değiştiği elle takip edilir.
- Bir header, linker script ya da bootloader yapılandırması değiştiğinde hangi çıktıların geçersiz olduğu gözden kaçabilir.
- Komutlar farklı terminallerde farklı bayraklarla çalıştırılabilir. Ortaya çıkan dosyanın nasıl üretildiği belirsizleşir.

Build sistemi bu iş akışını bir bağımlılık grafiği olarak yazar:

```text
kernel.c  ─┐
serial.c  ─┼─> nesne dosyaları ─┐
start.S   ─┘                    ├─> kernel.elf ─> boot imajı
linker.ld ───────────────────┘
boot yapılandırması ────────────────────> boot imajı
```

Bu grafikte bir kaynak dosyası değiştiğinde yalnızca ilgili nesne dosyası ve onu izleyen çıktılar yenilenir. Linker script değiştiğinde kaynaklar tekrar derlenmez, ancak kernel yeniden bağlanır. Build sisteminin temel görevi komutları kısaltmak değil, bu ilişkileri eksiksiz anlatmaktır.

## Make nasıl karar verir

Bir Make kuralı üç parçadan oluşur:

<!-- markdownlint-disable MD010 -->

```make
hedef: bağımlılıklar
	komut
```

<!-- markdownlint-enable MD010 -->

Hedef genellikle üretilecek bir dosyadır. Bağımlılıklar, o dosyayı üretmek için gereken girdilerdir. Girintili satır ise GNU Make dokümantasyonundaki adıyla recipe, yani çalıştırılacak komuttur. Komut satırının başında boşluk değil gerçek bir tab karakteri bulunur.

Make, hedef dosya yoksa veya bağımlılıklardan biri hedeften daha yeniyse komutu çalıştırır. Dosya içeriklerini karşılaştırmaz; son değiştirme zamanlarına bakar. Sistem saatinin geriye alınması ya da dosyaların zaman bilgileri korunarak kopyalanması bu nedenle yanlış bir “güncel” kararına yol açabilir.

Aşağıdaki kural, `src/kernel.c` değiştiğinde `build/kernel.o` dosyasını yeniler:

<!-- markdownlint-disable MD010 -->

```make
build/kernel.o: src/kernel.c
	mkdir -p build
	x86_64-elf-gcc -c src/kernel.c -o build/kernel.o
```

<!-- markdownlint-enable MD010 -->

Bu haliyle kural eksiktir. `kernel.c` içinde kullanılan bir header değiştiğinde Make bunu bilmez. Her header'ı elle listelemek mümkündür, fakat liste ilk unutulan `#include` satırında güvenilirliğini kaybeder. GCC'nin `-MMD` seçeneği, derleme sırasında bu ilişkileri bir `.d` dosyasına yazabilir. `-MP` ise silinen bir header için boş bir hedef ekleyerek Make'in daha anlaşılır biçimde devam etmesini sağlar.

## Küçük bir kernel için Makefile

Aşağıdaki örnek şu dizin düzenini varsayar:

```text
proje/
├── Makefile
├── linker.ld
├── include/
│   └── serial.h
└── src/
    ├── kernel.c
    ├── serial.c
    └── start.S
```

`start.S`, GNU assembler söz diziminde ve C ön işlemcisinden geçen bir assembly dosyasıdır. NASM kullanılan bir projede bunun için ayrı bir `nasm` kuralı gerekir.

<!-- markdownlint-disable MD010 -->

```make
.DEFAULT_GOAL := all

TARGET  ?= x86_64-elf
CC      := $(TARGET)-gcc
READELF := $(TARGET)-readelf

BUILD_DIR := build
KERNEL    := $(BUILD_DIR)/kernel.elf

C_SOURCES   := src/kernel.c src/serial.c
ASM_SOURCES := src/start.S
C_OBJECTS   := $(patsubst src/%.c,$(BUILD_DIR)/%.o,$(C_SOURCES))
ASM_OBJECTS := $(patsubst src/%.S,$(BUILD_DIR)/%.o,$(ASM_SOURCES))
OBJECTS     := $(C_OBJECTS) $(ASM_OBJECTS)
DEPS        := $(OBJECTS:.o=.d)

CPPFLAGS := -Iinclude -MMD -MP
CFLAGS   := -std=gnu11 -ffreestanding -mno-red-zone \
            -mgeneral-regs-only -mcmodel=kernel \
            -fno-stack-protector -fno-pic -fno-pie \
            -Wall -Wextra
ASFLAGS  := -ffreestanding -mno-red-zone -fno-pic -fno-pie
LDFLAGS  := -nostdlib -static -no-pie -T linker.ld \
            -Wl,-z,max-page-size=0x1000
LDLIBS   := -lgcc

.PHONY: all inspect clean

all: $(KERNEL)

$(KERNEL): $(OBJECTS) linker.ld Makefile
	$(CC) $(LDFLAGS) -o $@ $(OBJECTS) $(LDLIBS)

$(BUILD_DIR)/%.o: src/%.c Makefile
	mkdir -p $(@D)
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

$(BUILD_DIR)/%.o: src/%.S Makefile
	mkdir -p $(@D)
	$(CC) $(CPPFLAGS) $(ASFLAGS) -c $< -o $@

inspect: $(KERNEL)
	$(READELF) -h -l $(KERNEL)

clean:
	rm -rf -- build

-include $(DEPS)
```

<!-- markdownlint-enable MD010 -->

Kaynak listesi burada bilerek açık yazılmıştır. `wildcard` ile bütün `.c` dosyalarını otomatik toplamak daha kısa olabilir; buna karşılık yeni veya yanlış yerde duran bir dosyayı fark ettirmeden build'e dahil edebilir. Küçük projelerde açık liste, hangi kodun kernel'e girdiğini okumayı kolaylaştırır.

Örnekte sık kullanılan otomatik değişkenler şunlardır:

| Değişken | Değeri |
| --- | --- |
| `$@` | Kuralın hedefi |
| `$<` | İlk bağımlılık |
| `$^` | Tekrarları ayıklanmış bütün bağımlılıklar |
| `$(@D)` | Hedef dosyanın dizini |

`patsubst` ifadeleri kaynak adlarını `build/` altındaki nesne dosyalarına dönüştürür. `%` işareti aynı dosya adı gövdesini temsil eder. Böylece iki kaynak türü için tek tek kural yazmak yerine iki kalıp kural yeterli olur.

Nesne dosyalarının `Makefile`'a da bağlı olması bilinçli bir tercihtir. Derleyici bayrakları değiştiğinde Make yalnızca değişkenin eski değerini hatırlayamaz. `Makefile` bağımlılığı, dosyadaki herhangi bir değişiklikte nesneleri yeniden derleyerek geniş ama güvenli bir geçersiz kılma sağlar. Daha büyük projeler bayrakları ayrı damga dosyalarında tutarak bunu daraltabilir.

`-include $(DEPS)` satırı, GCC'nin ürettiği header bağımlılıklarını Makefile'a dahil eder. Baştaki `-`, ilk build sırasında henüz `.d` dosyaları yokken hata verilmesini engeller. `.DEFAULT_GOAL` açıkça belirtildiği için dahil edilen dosyalar varsayılan hedefi değiştiremez.

Bağlama GCC sürücüsüyle yapıldığı için uygun `libgcc` dosyası `-lgcc` ile bulunur. Kütüphaneler komut satırında nesne dosyalarından sonra yer alır. Buradaki bayrakların nedeni ve red zone kullanmayan `libgcc` gereksinimi [cross-compiler notunda](cross-compiler.md#kernel-derlemesinde-kullanılan-bayraklar) ayrıntılı olarak ele alınır.

Bu Makefile bir `kernel.elf` üretir; tek başına boot edilebilir bir disk ya da ISO üretmez. Seçilen bootloader'la ilgili dosya kopyalama ve imaj oluşturma adımları, kernel'i bağlayan kuraldan sonra gelen ayrı bir dosya hedefi olmalıdır.

## İmaj ve çalıştırma hedefleri

Boot imajı gerçek bir dosyadır; bu yüzden `.PHONY` olarak işaretlenmemelidir. İmajı üreten kural en azından kernel'e, bootloader yapılandırmasına ve kullandığı betiğe bağlı olmalıdır:

<!-- markdownlint-disable MD010 -->

```make
IMAGE := $(BUILD_DIR)/os.iso

$(IMAGE): $(KERNEL) boot/limine.conf scripts/make-image.sh
	scripts/make-image.sh $(KERNEL) $@

.PHONY: image run

image: $(IMAGE)

run: $(IMAGE)
	qemu-system-x86_64 -cdrom $(IMAGE)
```

<!-- markdownlint-enable MD010 -->

Buradaki `make-image.sh` yalnızca proje tarafından sağlanacak arayüzü gösterir. Betiğin içeriği ve QEMU seçenekleri, kullanılan boot protokolüne ve imaj biçimine göre yazılır. Önemli nokta, betiğin kendi başına bütün projeyi yeniden derlememesi ve girdilerinin Make kuralında görünmesidir.

`image` ve `run` dosya adı değil, kullanıcının istediği eylemlerdir. `.PHONY` bildirimi, proje kökünde bu adlarda bir dosya bulunsa bile komutların doğru çalışmasını sağlar. Gerçek bir dosya hedefi phony yapılırsa Make onu her seferinde yeniden üretir ve artımlı build'in yararı kaybolur.

## Günlük kullanım ve teşhis

Varsayılan hedefi derlemek için yalnızca `make` yeterlidir. Birden fazla işi paralel yürütmek için iş sayısı komut satırından verilir:

```sh
make -j"$(nproc)"
```

Makefile içinde yeniden `make -j` çağırmak yerine paralellik kararını kullanıcıya bırakmak gerekir. Alt dizinlerde Make kullanılıyorsa `make` komutunun sabit adı yerine `$(MAKE)` yazılır; böylece GNU Make'in jobserver mekanizması paralel iş sayısını alt süreçlerle paylaşabilir.

Bir hedefin hangi komutları çalıştıracağını, komutları gerçekten çalıştırmadan görmek için:

```sh
make -n
```

Make'in neden bir hedefi yenilediğini izlemek için:

```sh
make --trace
```

Derleme sonucunun hedef mimarisini ve ELF bölümlerini denetlemek için örnekteki `make inspect` hedefi kullanılabilir. `make clean && make` ise günlük build komutu haline getirilmemelidir. Temiz build'in çalışması, bağımlılık grafiğinin doğru olduğunu kanıtlamaz; artımlı build'deki eksikleri gizler.

Makefile bir kabuk betiği değildir. Varsayılan olarak her komut satırı ayrı bir kabukta çalışır. Bu nedenle aşağıdaki iki satırda `cd` ikinci satırı etkilemez:

<!-- markdownlint-disable MD010 -->

```make
yanlis:
	cd build
	pwd
```

<!-- markdownlint-enable MD010 -->

Aynı kabuk durumunu paylaşması gereken komutlar tek satırda birleştirilebilir veya betiğe taşınabilir. Uzun imaj hazırlama işlemlerini ayrı bir betikte tutmak, Makefile'ın bağımlılık grafiği olarak okunmasını kolaylaştırır.

## Tekrarlanabilir ve paralel build

Make aynı komutları çalıştırır, ancak araçların sürümünü veya ortamı kendiliğinden sabitlemez. Tekrarlanabilir bir build için proje şunları açık tutmalıdır:

- Hedef üçlüsü ve gereken toolchain sürümü dokümante edilmelidir.
- Derleyici, assembler ve linker host araçlarına düşmeden hedef ön ekiyle çağrılmalıdır.
- Üretilen bütün dosyalar `build/` gibi ayrı bir dizinde tutulmalı, kaynak ağacına yazılmamalıdır.
- Debug ve release gibi farklı bayrak kümeleri aynı nesne dosyalarını paylaşmamalıdır. `build/debug/` ve `build/release/` gibi ayrı dizinler kullanılabilir.
- Bir kural, hedefini üretmeden başarılı dönmemelidir. Yarım kalan dosya sonraki build'de güncel sanılmamalıdır.

Paralel build için her kural yalnızca kendi hedefini üretmeli ve kullandığı her girdiyi bağımlılık olarak bildirmelidir. İki kuralın aynı geçici dosyaya yazması, seri build'de fark edilmeyip `make -j` altında ara sıra bozulan sonuçlara yol açar. Paralel derleme burada yalnızca hızlandırma değil, eksik bağımlılıkları ortaya çıkaran bir doğrulama aracıdır.

## Alternatifler

| Araç | Güçlü yanı | Dikkat edilmesi gereken |
| --- | --- | --- |
| GNU Make | Her Unix benzeri ortamda bulunur, küçük projede grafik doğrudan okunabilir | Söz diziminin tarihsel ayrıntıları vardır; büyük Makefile'lar zor bakım görür |
| Ninja | Bağımlılık grafiğini çok hızlı çalıştırır | Genellikle elle yazılmak için değil, başka bir araç tarafından üretilmek için tasarlanmıştır |
| CMake | Birden çok host ve editör için Makefile ya da Ninja dosyası üretebilir | Bare metal hedef için toolchain dosyası ve derleyici denemeleri ayrıca yapılandırılmalıdır |
| Meson | Cross file ile host ve target araçlarını ayırır, Ninja ile hızlı build sunar | Özel boot imajı adımları yine proje tarafından tanımlanır |
| Kabuk betiği | İlk prototipte doğrusal bir iş akışını yazmak kolaydır | Artımlı build, paralellik ve bağımlılık takibi yeniden yazılmak zorunda kalır |

Küçük ve eğitim amaçlı bir kernel için Make genellikle yeterlidir. Proje birden çok mimariyi, yapılandırma seçeneğini ve host sistemini desteklemeye başladığında CMake ya da Meson yapılandırma katmanını daha düzenli tutabilir. Ninja ise bu araçların ürettiği alt katman olarak anlam kazanır.

CMake kullanılıyorsa cross-compiler bir toolchain dosyasında tanımlanır. Normal bir kullanıcı programını bağlamayı deneyen yapılandırma kontrolleri, linker script olmadan başarısız olabilir. `CMAKE_TRY_COMPILE_TARGET_TYPE` değerini `STATIC_LIBRARY` yapmak bu kontrollerde executable bağlanmasını engeller. Meson'daki karşılık, derleyici ve hedef makine bilgilerinin tutulduğu cross file'dır.

Araç seçimi kernel'in mimarisini belirlemez. Önce çıktılar ve aralarındaki bağımlılıklar doğru modellenmeli, ancak mevcut araç bu grafiği yönetmekte gerçekten zorlanıyorsa daha yüksek seviyeli bir sisteme geçilmelidir.

## Sık yapılan hatalar

- Header bağımlılıklarını izlememek. Header değişir, eski nesne dosyası link'e girmeye devam eder.
- `kernel.elf` veya boot imajı gibi gerçek dosyaları `.PHONY` yapmak. Bu dosyalar her `make` çağrısında gereksiz yere yeniden üretilir.
- `clean`, `run` ve `inspect` gibi eylem hedeflerini `.PHONY` yapmamak. Aynı adlı bir dosya oluştuğunda hedef sessizce çalışmaz.
- Make komutlarını boşlukla girintilemek. Varsayılan söz diziminde komut satırı tab ile başlar.
- İmaj kuralına kernel'i, boot yapılandırmasını veya imaj betiğini bağımlılık olarak eklememek.
- Her build'den önce `clean` çalıştırmak. Bu alışkanlık bozuk artımlı build'i gizler ve gereksiz zaman harcatır.
- `run` hedefini kernel dosyasını üreten komutların arasına koymak. Derleme ve emülatörü çalıştırma ayrı hedefler olmalıdır.
- Paralel kuralların aynı geçici dosyayı kullanması. Hata yalnızca yük altında ve düzensiz aralıklarla görünür.
- Derlemeyi cross-compiler ile yapıp bağlama veya inceleme adımında fark etmeden host aracına dönmek.

## İlgili notlar

- [Geliştirme ortamı kurulumu](gelistirme-ortami.md) — GNU Make ve diğer araçların kurulumu
- [Cross-compiler hazırlama](cross-compiler.md) — hedef üçlüsü, derleyici bayrakları ve `libgcc`
- [Limine](../02-boot/limine.md) — kernel'i boot edilebilir bir imaja yerleştirme
- [02 — Boot](../02-boot/) — boot protokolleri ve imaj yapısı

## Kaynaklar

- GNU Make Manual, Bölüm 2 — An Introduction to Makefiles; <https://www.gnu.org/software/make/manual/make.html> (erişim: 22 Eylül 2026)
- GNU Make Manual, Bölüm 4.3 — Types of Prerequisites ve Bölüm 4.6 — Phony Targets; <https://www.gnu.org/software/make/manual/html_node/Rules.html> (erişim: 22 Eylül 2026)
- GNU Make Manual, Bölüm 4.14 — Generating Prerequisites Automatically; <https://www.gnu.org/software/make/manual/html_node/Automatic-Prerequisites.html> (erişim: 22 Eylül 2026)
- GCC Manual, “Options Controlling the Preprocessor” (`-MMD`, `-MP`); <https://gcc.gnu.org/onlinedocs/gcc/Preprocessor-Options.html> (erişim: 22 Eylül 2026)
- Ninja Manual, “Conceptual overview”; <https://ninja-build.org/manual.html> (erişim: 22 Eylül 2026)
- CMake dokümantasyonu, `cmake-toolchains(7)` — Cross Compiling; <https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html> (erişim: 22 Eylül 2026)
- Meson dokümantasyonu, “Cross compilation”; <https://mesonbuild.com/Cross-compilation.html> (erişim: 22 Eylül 2026)
