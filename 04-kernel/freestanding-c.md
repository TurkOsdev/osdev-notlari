# Freestanding C ortamı ve kısıtları

Normal bir C programı bir işletim sisteminin üzerinde çalışır. `printf`, `malloc` ve `fopen` gibi fonksiyonlar bu işletim sistemine dayanan standart kütüphaneden gelir. Kernel ise bu işletim sisteminin kendisidir, altında onu destekleyen bir şey yoktur. C standardı bu durumu freestanding ortam olarak tanımlar. Freestanding ortamda yalnızca birkaç header kullanılabilir. Geri kalan her şeyi kernel kendisi yazar.

## Ön koşullar

- [Cross-compiler hazırlama](../00-giris/cross-compiler.md)

## Kapsam

Bu not C11 standardını ve GCC'yi esas alır. Derleyici bayrakları x86-64 içindir.

## Problem

Kernel'de standart kütüphane kullanılamaz, çünkü standart kütüphane işini system call'larla yapar. Örneğin `printf` sonunda bir `write` system call'ı yapar. Kernel'in system call yapacağı başka bir kernel yoktur. Bu yüzden derleyiciye kodun freestanding olduğu söylenmeli ve hangi parçaların kullanılabileceği bilinmelidir.

## Hosted ve freestanding

C standardı iki tür ortam tanımlar:

- Hosted: bütün standart kütüphane vardır ve program `main` ile başlar.
- Freestanding: yalnızca belirli header'lar vardır. Programın nereden başlayacağını ortam belirler.

C11'de freestanding ortamda bulunması gereken header'lar şunlardır:

| Header | İçerik |
| --- | --- |
| `<stddef.h>` | `size_t`, `NULL`, `offsetof` |
| `<stdint.h>` | `uint8_t`, `uint64_t` gibi sabit boyutlu tipler |
| `<stdbool.h>` | `bool`, `true`, `false` |
| `<stdarg.h>` | değişken sayıda argüman (`va_list`) |
| `<limits.h>` | tiplerin sınırları |
| `<float.h>` | ondalıklı sayı tiplerinin özellikleri |
| `<stdalign.h>` | `alignas`, `alignof` |
| `<stdnoreturn.h>` | `noreturn` |
| `<iso646.h>` | `and`, `or` gibi alternatif yazımlar |

Bu header'lar yalnızca tip ve makro tanımlar, çalışan kod içermez. Bu yüzden işletim sistemi olmadan da kullanılabilirler.

## Derleyici tarafında

GCC'ye freestanding derleme yapılacağı `-ffreestanding` ile söylenir. Bu bayrak derleyicinin standart fonksiyonlar hakkında varsayım yapmasını azaltır. Kernel derlerken sık kullanılan bayraklar:

| Bayrak | Neden |
| --- | --- |
| `-ffreestanding` | Freestanding ortam olduğunu bildirir |
| `-nostdlib` | Bağlama sırasında standart kütüphaneyi ve başlangıç dosyalarını eklemez |
| `-fno-stack-protector` | Stack koruma kodu çalışma zamanı desteği ister |
| `-mno-red-zone` | Interrupt'ların stack'i bozmasını önler |
| `-mgeneral-regs-only` | SSE ve diğer ondalıklı sayı register'larının kullanılmasını engeller |

Ayrıntılar için [cross-compiler notuna](../00-giris/cross-compiler.md) bakılabilir.

## Implementasyon notları

- GCC, freestanding modda bile `memcpy`, `memmove`, `memset` ve `memcmp` fonksiyonlarına çağrı üretebilir. Örneğin büyük bir `struct` kopyalaması `memcpy` çağrısına dönüşebilir. Bu dört fonksiyonu kernel kendisi sağlamalıdır.
- Bazı işlemler için GCC, `libgcc` içindeki yardımcı fonksiyonları çağırır. Bu yüzden kernel bağlanırken `-lgcc` eklenir.
- Kernel'in giriş noktası `main` değildir. Bootloader'ın atladığı sembol, örneğin `_start` veya `kernel_main`, linker script ile belirlenir.
- Global değişkenlerin constructor'ları (C++ veya `__attribute__((constructor))`) kendiliğinden çalışmaz. Gerekirse kernel bunları kendisi çağırır.
- `printf` yerine seri port'a veya ekrana yazan küçük bir `kprintf` yazmak, ilk yapılan işlerden biridir. Bunun için `<stdarg.h>` yeterlidir.

```c
#include <stddef.h>

/* GCC'nin çağırabileceği fonksiyonlardan biri. */
void *memset(void *hedef, int deger, size_t n)
{
    unsigned char *p = hedef;
    while (n--)
        *p++ = (unsigned char)deger;
    return hedef;
}
```

Bu örnek basitlik için bayt bayt çalışır, hızlı değildir.

## Sık yapılan hatalar

- Kernel kodunda `<stdio.h>` veya `<stdlib.h>` eklemek. Cross-compiler kullanılıyorsa bu header'lar zaten bulunmaz.
- Host derleyicisiyle derlemek. Host'un header'ları ve kütüphaneleri sessizce kernel'e karışabilir.
- `memcpy` ve `memset` yazmayı unutmak. Bağlama aşamasında "undefined reference" hatası alınır.
- `-mgeneral-regs-only` olmadan `float` ve `double` kullanmak. SSE register'ları context switch'te kaydedilmiyorsa değerler bozulur.

## İlgili notlar

- [Cross-compiler hazırlama](../00-giris/cross-compiler.md)
- [Build sistemi](../00-giris/build-sistemi.md)
- [Stack ve stack frame](../03-assembly/stack.md)

## Kaynaklar

- ISO/IEC 9899:2011 (C11), Bölüm 4: Conformance (freestanding header listesi)
- ISO/IEC 9899:2011 (C11), Bölüm 5.1.2.1: Freestanding environment
- GCC Manual, "C Language Standards" (freestanding ortamda gereken `memcpy`, `memmove`, `memset`, `memcmp`); <https://gcc.gnu.org/onlinedocs/gcc/Standards.html> (erişim: 16 Ekim 2026)
- GCC Manual, "Options Controlling C Dialect" (`-ffreestanding`); <https://gcc.gnu.org/onlinedocs/gcc/C-Dialect-Options.html> (erişim: 20 Ekim 2026)
- GCC Internals, "The GCC low-level runtime library" (`libgcc`); <https://gcc.gnu.org/onlinedocs/gccint/Libgcc.html> (erişim: 16 Ekim 2026)
