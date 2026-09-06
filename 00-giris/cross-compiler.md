# Cross-compiler hazırlama

Bir derleyici, hedef makine hakkında yapılandırma sırasında karar verilmiş varsayımlarla gelir: hangi nesne dosyası formatını üreteceği, hangi ABI'yi izleyeceği, hangi başlıkları arayacağı ve programın başlangıcında hangi kodun çalışacağı. Dağıtımla gelen GCC bu varsayımları o dağıtımın kullanıcı programları için yapar; ürettiği nesne dosyası bir libc'ye, bir dinamik yükleyiciye ve altında çalışan bir kernel'e bağlıdır. Kernel'in bağlanacağı böyle bir katman yoktur.

Cross-compiler, host (üzerinde derleme yapılan sistem) ile target (kodun çalışacağı sistem) ayrımını yapılandırma zamanında sabitleyen bir toolchain'dir. `x86_64-elf` hedefine kurulmuş bir GCC'nin bir libc'si, bir crt başlangıç kodu ve bir dinamik yükleyicisi yoktur; bu yüzden bunlara ait varsayımlar komut satırında bir bayrak unutulduğunda geri sızamaz. Kazanç, bayraklardan tasarruf etmek değil, host'un varsayılanlarının derleme sonucunu sessizce değiştirememesidir.

## Ön koşullar

- [Geliştirme ortamı kurulumu](gelistirme-ortami.md)
- [Ön gereksinimler](on-gereksinimler.md)

## Kapsam

Bu not x86-64 hedefini, GNU toolchain'ini (binutils + GCC) ve LLVM/Clang'i esas alır. Komutlar Debian/Ubuntu üzerinde yazılmıştır; paket adları dışında diğer dağıtımlarda da aynıdır. Arşiv adlarındaki sürüm numaralarını indirdiğiniz sürümle değiştirin.

## Problem

Host derleyicisiyle kernel derlemeyi denediğinizde ortaya çıkan sorunlar tek bir başlık altında toplanmaz; birbirinden bağımsız birkaç varsayımdan gelir.

**Başlangıç kodu ve kütüphane.** Host GCC bağlama sırasında `crt1.o`, `crti.o`, `crtn.o` ve libc'yi otomatik ekler. `crt1.o` içindeki `_start`, libc'yi kurup `main`'i çağırır. Kernel'de `main` yoktur, bağlayacak libc yoktur ve giriş noktasını linker script belirler.

**Başlıklar.** `-I` verilmese bile host'un `/usr/include` dizini arama yoluna dahildir. `#include <stdio.h>` satırının derlenmesi, o kodun çalışacağı anlamına gelmez.

**Dağıtım varsayılanları.** Birçok dağıtım GCC'yi `--enable-default-pie` ve `-fstack-protector-strong` varsayılanlarıyla derler. Birincisi kernel'in beklemediği bir konumdan bağımsız çıktı üretir, ikincisi `__stack_chk_fail` ve `__stack_chk_guard` sembollerine bağımlılık ekler. Bu varsayılanlar dağıtımdan dağıtıma ve sürümden sürüme değişir; kernel'in derlenip derlenmemesi böyle bir farka bağlı kalmamalıdır.

**Format ve mimari.** Host x86-64 Linux değilse sorun varsayımlarla sınırlı kalmaz. macOS'ta native format Mach-O'dur ve sistemin `ld` komutu ELF üretmez. `x86_64` bir host'tan `i686` veya `aarch64` hedeflemek de aynı gruba girer.

`-ffreestanding`, `-nostdlib`, `-fno-pie` ve `-fno-stack-protector` bayrakları bu farkların bir kısmını kapatır. x86-64 Linux üzerinde x86-64 kernel derleyen projelerin host derleyicisiyle çalışması bu yüzden mümkündür; Linux kernel'i de böyle derlenir. Ancak bu, uzun ve eksiksiz tutulması gereken bir bayrak listesine bağımlılık demektir ve host ile target'ın mimarisi ayrıştığı anda tümüyle geçersiz kalır.

## Hedef üçlüsü

Toolchain'in hangi sistem için üretim yapacağı, hedef üçlüsü (target triple) denen bir adla belirtilir. Ad `mimari-satıcı-sistem-abi` alanlarından oluşur ve alanların bir kısmı atlanabilir:

| Üçlü | Anlamı |
| --- | --- |
| `x86_64-elf` | x86-64, işletim sistemi yok, çıktı ELF |
| `i686-elf` | 32 bit x86, işletim sistemi yok |
| `aarch64-none-elf` | ARM64, işletim sistemi yok |
| `riscv64-unknown-elf` | RISC-V 64 bit, işletim sistemi yok |
| `x86_64-linux-gnu` | x86-64 üzerinde Linux ve glibc |

Kernel için `-elf` ile biten üçlü seçilir. `-linux-gnu` üçlüsü glibc'nin varlığını, Linux'un system call ABI'sini ve bir dinamik yükleyiciyi ima eder; bunların hiçbiri hedef sistemde yoktur.

GNU toolchain'inde araçlar üçlüyle ön eklenmiş adlarla kurulur: `x86_64-elf-gcc`, `x86_64-elf-ld`, `x86_64-elf-objdump`. Bu adlandırma, host araçlarıyla karışmayı önler. Clang'da ayrı bir binary yoktur; üçlü her çağrıda `--target=` ile verilir.

## Seçenekler

| Yöntem | Uygun olduğu durum | Dikkat edilmesi gereken |
| --- | --- | --- |
| Dağıtım veya Homebrew paketi | Hedef üçlüsü için hazır paket varsa | Paketin hangi seçeneklerle derlendiği her zaman belgelenmez |
| Kaynaktan derleme | Yapılandırmanın denetlenmesi gerekiyorsa | Bir kez uzun sürer, sonucu taşınabilirdir |
| Clang + LLD | Birden çok mimari hedefleniyorsa | libgcc karşılığı ayrıca ele alınmalıdır |

Hazır paket bulunabiliyorsa denemeye değer; `-dumpmachine` çıktısı beklenen üçlüyü veriyorsa çoğu proje için yeterlidir. Kaynaktan derleme, red zone'suz libgcc gibi yapılandırmaya müdahale gerektiren durumlarda zorunlu hale gelir.

## Kaynaktan GNU toolchain

Kurulum host'un `/usr` ağacına dokunmayan bir dizine yapılır. Aşağıdaki değişkenler, kurulum boyunca açık kalan kabukta tanımlı olmalıdır:

```sh
export PREFIX="$HOME/opt/cross"
export TARGET=x86_64-elf
export PATH="$PREFIX/bin:$PATH"
```

`PATH` satırı isteğe bağlı değildir: GCC'nin yapılandırması ve derlenmesi sırasında hedefe ait assembler ve linker (`$TARGET-as`, `$TARGET-ld`) aranır. Bunlar bulunamazsa GCC derlenir ancak kullanılamaz bir halde kurulur.

Önce binutils. Kaynak ağacının içinde değil, ayrı bir derleme dizininde yapılandırılır:

```sh
mkdir build-binutils && cd build-binutils
../binutils-2.44/configure --target="$TARGET" --prefix="$PREFIX" \
    --with-sysroot --disable-nls --disable-werror
make -j"$(nproc)"
make install
```

Sonra GCC:

```sh
mkdir build-gcc && cd build-gcc
../gcc-15.1.0/configure --target="$TARGET" --prefix="$PREFIX" \
    --disable-nls --enable-languages=c --without-headers
make -j"$(nproc)" all-gcc
make -j"$(nproc)" all-target-libgcc
make install-gcc
make install-target-libgcc
```

Yapılandırma seçeneklerinin anlamı:

| Seçenek | Ne yapar |
| --- | --- |
| `--target` | Üretilecek kodun hedef üçlüsü |
| `--prefix` | Kurulum dizini; host araçlarından ayrı tutulur |
| `--with-sysroot` | binutils'e sysroot desteğini derler, ilerideki libc çalışması için gerekir |
| `--disable-nls` | Çevrilmiş mesajları kapatır, bağımlılığı azaltır |
| `--disable-werror` | Uyarıların derlemeyi durdurmasını engeller |
| `--enable-languages=c` | Yalnızca C derleyicisi; süreyi belirgin biçimde kısaltır |
| `--without-headers` | Hedefte libc başlığı bulunmadığını bildirir |

GCC'de düz `make` çalıştırılmaz. Varsayılan hedef, target için libc gerektiren kütüphaneleri de derlemeye çalışır ve `--without-headers` ile yapılandırılmış bir ağaçta hata verir. Bu yüzden yalnızca `all-gcc` ve `all-target-libgcc` hedefleri istenir.

Kurulum bittiğinde `$PREFIX/bin` dizininin `PATH` içinde kalıcı olması gerekir; kabuk yapılandırma dosyanıza ekleyin.

## libgcc

`--enable-languages=c` ile kurulmuş bir derleyici bile bazı işlemleri tek komuta indiremez ve bunlar için çağrı üretir: 128 bit tamsayı bölme (`__udivti3`), 32 bit hedefte 64 bit aritmetik, taşma denetimli çarpma, donanım desteği olmayan kayan nokta işlemleri. Bu işlevler libgcc içindedir ve libc'den bağımsızdır. Bağlama sırasında `-lgcc` verilmezse hata, kaynak kodda hiç görünmeyen bir sembolde çıkar:

```text
undefined reference to `__udivti3'
```

`-lgcc` bağımsız değişkeni, nesne dosyalarından **sonra** yazılır; linker arşivleri komut satırı sırasına göre tarar.

**Red zone.** x86-64 System V ABI, `rsp`'nin altındaki 128 baytlık bölgeyi (red zone) yaprak fonksiyonların serbestçe kullanabileceği alan olarak tanımlar. Tanım, bu bölgeyi asenkron olayların bozmayacağı varsayımına dayanır. Bare metal ortamda böyle bir garanti yoktur: interrupt geldiğinde işlemci `rsp`'nin gösterdiği yerden itibaren yazmaya başlar ve red zone'daki veriyi ezer. Kendi kodunuzu `-mno-red-zone` ile derlemek yeterli değildir, çünkü kernel'in çağırdığı libgcc işlevleri red zone kullanılarak derlenmiştir.

Çözüm, libgcc'nin `-mno-red-zone` sürümünü de üreten bir multilib tanımlamaktır. GCC kaynak ağacında `gcc/config/i386/t-x86_64-elf` adında bir dosya oluşturulur:

```make
MULTILIB_OPTIONS += mno-red-zone
MULTILIB_DIRNAMES += no-red-zone
```

`gcc/config.gcc` içinde `x86_64-*-elf*` durumuna bu dosya eklenir:

```text
tmake_file="${tmake_file} i386/t-x86_64-elf"
```

Bu değişiklik yapılandırmadan önce yapılmalıdır. Sonrasında `-mno-red-zone` ile derlenen kod, libgcc'nin doğru sürümüne otomatik olarak bağlanır. Doğrulaması `x86_64-elf-gcc -mno-red-zone -print-libgcc-file-name` çıktısındaki yolun `no-red-zone` dizinini içermesidir.

## Clang ile

Clang tek kurulumda tüm hedefleri destekler; ayrı bir toolchain derlemek gerekmez. Hedef her çağrıda verilir:

```sh
clang --target=x86_64-elf -ffreestanding -mno-red-zone -c kernel.c -o kernel.o
ld.lld -T linker.ld -o kernel.elf kernel.o
```

Karşılığında iki noktanın ayrıca ele alınması gerekir. Birincisi, binutils araçlarının yerini `llvm-objcopy`, `llvm-objdump`, `llvm-readelf` ve `llvm-nm` alır; bunlar çoğu kullanımda uyumludur ancak bayrakları birebir aynı değildir. İkincisi, libgcc'nin karşılığı olan compiler-rt builtins kütüphanesi bare metal hedefler için genellikle paketlenmez. Ya `compiler-rt`'nin builtins bileşenini hedef için ayrıca derlemek ya da o çağrıları gerektiren yapıları (128 bit bölme gibi) kullanmamak gerekir.

Clang'ın avantajı, birden çok mimariyi hedefleyen bir projede tek toolchain'in yetmesidir. GNU tarafında her mimari için ayrı bir cross-compiler derlenir.

## Kurulumun doğrulanması

Sürüm çıktısı, derleyicinin doğru hedefe kurulduğunu göstermez. Üçlüyü derleyicinin kendisine sorun:

```sh
x86_64-elf-gcc -dumpmachine          # x86_64-elf
x86_64-elf-gcc -print-libgcc-file-name
x86_64-elf-ld --version
```

Ardından freestanding bir çeviri biriminin gerçekten derlendiğini doğrulayın:

```c
// deneme.c — toolchain doğrulaması. Kernel giriş noktası değildir.
#include <stdint.h>

__attribute__((noreturn)) void kernel_main(void) {
    volatile uint16_t *ekran = (volatile uint16_t *)0xB8000;
    ekran[0] = 0x0F00 | 'T';
    for (;;) {
        __asm__ volatile ("hlt");
    }
}
```

```sh
x86_64-elf-gcc -ffreestanding -mno-red-zone -mcmodel=kernel \
    -fno-stack-protector -c deneme.c -o deneme.o
x86_64-elf-readelf -h deneme.o
```

`readelf` çıktısında `Machine` alanı `Advanced Micro Devices X86-64`, `Type` alanı `REL` görünmelidir. Örnek yalnızca derlemeyi sınar: `0xB8000` adresindeki VGA metin tamponu long mode'da ve identity mapping varsayımı altında geçerlidir, bu koşulları sağlamak bootloader'ın ve kernel'in erken kodunun işidir.

## Kernel derlemesinde kullanılan bayraklar

Cross-compiler, host varsayımlarını ortadan kaldırır; hedef donanımın kısıtlarını bildirmek yine kod üretimi bayraklarının işidir.

| Bayrak | Neden |
| --- | --- |
| `-ffreestanding` | Standart kütüphanenin varlığını varsaymaz, `main` özel muamelesi görmez |
| `-nostdlib` | Bağlama sırasında crt dosyalarını ve libc'yi eklemez |
| `-mno-red-zone` | Interrupt'ın ezeceği bölgenin kullanılmasını engeller |
| `-mgeneral-regs-only` | SSE/MMX register'larının kullanılmasını engeller |
| `-mcmodel=kernel` | Kernel'i adres alanının üst 2 GiB'ına yerleştiren kod modeli |
| `-fno-stack-protector` | `__stack_chk_fail` bağımlılığını kaldırır |
| `-fno-pie -no-pie` | Sabit adrese bağlanan çıktı üretir |
| `-fno-strict-aliasing` | Tip kuralına dayanan optimizasyonları kapatır |
| `-fno-delete-null-pointer-checks` | Adres 0'ın erişilemez olduğu çıkarımını engeller |
| `-Wl,-z,max-page-size=0x1000` | ELF hizalamasını 2 MiB'dan 4 KiB'a indirir, imaj boyutunu küçültür |

SSE ile ilgili bayrak gerekir, çünkü x86-64'te derleyici izin verildiğinde SSE komutlarını serbestçe üretir; oysa SSE kullanımı `CR0` ve `CR4` içindeki bitlerin ayarlanmasını ve bağlam değişiminde register durumunun saklanmasını gerektirir. Bu altyapı kurulmadan üretilen ilk SSE komutu exception ile sonuçlanır.

Assembly dosyalarında `.note.GNU-stack` bölümünün bulunmaması, binutils 2.39 ve sonrasında linker uyarısına yol açar. Her assembly dosyasına aşağıdaki satırı eklemek veya bağlama sırasında `-z noexecstack` vermek uyarıyı giderir:

```asm
; NASM söz dizimi
section .note.GNU-stack noalloc noexec nowrite progbits
```

## Mimariye göre farklar

| Mimari | Hedef üçlüsü ve dikkat edilecek noktalar |
| --- | --- |
| x86-64 | `x86_64-elf`; red zone kapatılır, SSE kapatılır, `-mcmodel=kernel` |
| ARM64 | `aarch64-none-elf`; `-mgeneral-regs-only` ile FP/SIMD kapatılır, kod modeli `-mcmodel=large` ile seçilir |
| RISC-V | `riscv64-unknown-elf`; `-march` ile uzantı kümesi (`rv64imac`), `-mabi=lp64` ile yumuşak kayan nokta ABI'si seçilir |

RISC-V'de `-march` ve `-mabi` birbirine uyumlu olmak zorundadır: `lp64d` ABI'si donanım çift duyarlıklı kayan nokta gerektirir, `lp64` gerektirmez. Uyumsuz seçim bağlama aşamasında ABI çakışması hatası verir.

## Sık yapılan hatalar

- `$PREFIX/bin` dizinini `PATH`'e eklemeden GCC'yi yapılandırmak. Derleme tamamlanır, kurulan derleyici hedefe ait assembler'ı bulamaz.
- GCC ağacında düz `make` çalıştırmak. `all-gcc` ve `all-target-libgcc` dışındaki hedefler libc arar.
- Bağlama satırına `-lgcc` yazmamak veya nesne dosyalarından önce yazmak.
- Red zone kullanan libgcc ile çalışmak. Hata, interrupt'ın hangi anda geldiğine bağlı olarak düzensiz aralıklarla ortaya çıkar.
- `-ffreestanding` verip `-nostdlib` unutmak. Host'un `crt1.o` dosyası bağlanır ve `_start` sembolü çakışır.
- Derlemeyi cross-compiler ile yapıp bağlamayı host `ld` komutuyla yapmak.
- Hedef üçlüsü olarak `x86_64-linux-gnu` seçmek. Bu üçlü Linux ve glibc varsayar.
- Toolchain'i `/usr` altına kurmak ve dağıtımın paketleriyle karıştırmak.
- Derleyiciyi yükselttikten sonra libgcc'yi yeniden derlememek.

## İlgili notlar

- [Geliştirme ortamı kurulumu](gelistirme-ortami.md) — paketler ve emülatör kurulumu
- [Ön gereksinimler](on-gereksinimler.md) — freestanding C ve ABI bilgisi
- [Nereden başlanır: yol haritası](yol-haritasi.md) — Aşama 0
- [02 — Boot](../02-boot/) — üretilen imajın önyüklenmesi

## Kaynaklar

- GCC, "Installing GCC: Configuration"; <https://gcc.gnu.org/install/configure.html> (erişim: 7 Eylül 2026)
- GCC Manual, Bölüm 3.19.60 — x86 Options; <https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html> (erişim: 7 Eylül 2026)
- GCC Manual, Bölüm 3.18 — Options for Code Generation Conventions; <https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html> (erişim: 7 Eylül 2026)
- GCC Internals, "The GCC low-level runtime library"; <https://gcc.gnu.org/onlinedocs/gccint/Libgcc.html> (erişim: 7 Eylül 2026)
- GNU Binutils, "ld: Command-line Options"; <https://sourceware.org/binutils/docs/ld/> (erişim: 7 Eylül 2026)
- System V Application Binary Interface — AMD64 Architecture Processor Supplement, Bölüm 3.2.2 (The Stack Frame); <https://gitlab.com/x86-psABIs/x86-64-ABI> (erişim: 7 Eylül 2026)
- Clang dokümantasyonu, "Cross-compilation using Clang"; <https://clang.llvm.org/docs/CrossCompilation.html> (erişim: 7 Eylül 2026)
- LLVM, "compiler-rt builtins"; <https://compiler-rt.llvm.org/> (erişim: 7 Eylül 2026)
- RISC-V ELF psABI Specification; <https://github.com/riscv-non-isa/riscv-elf-psabi-doc> (erişim: 7 Eylül 2026)
- OSDev Wiki, "GCC Cross-Compiler"; <https://wiki.osdev.org/GCC_Cross-Compiler> (erişim: 7 Eylül 2026)
- OSDev Wiki, "Libgcc without red zone"; <https://wiki.osdev.org/Libgcc_without_red_zone> (erişim: 7 Eylül 2026)
