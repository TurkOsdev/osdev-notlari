# Endianness

Endianness, birden fazla bayttan oluşan bir değerin bellekte hangi sırayla tutulduğunu anlatır. 32 bitlik `0x12345678` değeri dört bayttır. Bu dört baytın hangisinin düşük adrese yazılacağı işlemciye göre değişir.

İki yaygın düzen vardır. Little endian düzende en düşük anlamlı bayt en düşük adrese yazılır. Big endian düzende ise en yüksek anlamlı bayt en düşük adrese yazılır.

## Kapsam

Bu not x86-64 üzerinden anlatılır. ARM64 ve RISC-V farkları ayrı bir bölümde verilmiştir.

## Problem

İşlemci kendi içinde değerleri hep aynı düzende okuyup yazdığı için tek başına çalışan bir programda endianness genelde fark edilmez. Sorun, veri başka bir yerden geldiğinde ortaya çıkar:

- Ağ paketlerinin başlıkları big endian olarak gönderilir.
- Disk üzerindeki dosya sistemi yapılarının bir düzeni vardır. ext2 ve FAT little endian kullanır.
- Bir aygıtın register'ları belirli bir düzende okunmak üzere tasarlanmış olabilir.

Bu verileri düzeni bilmeden okuyan kod yanlış sayılar elde eder. Örneğin ağdan gelen 80 numaralı port, little endian bir makinede düzeltilmeden okunursa 20480 olarak görülür.

## Donanım seviyesinde

`0x12345678` değeri `0x1000` adresine yazıldığında baytlar şöyle yerleşir:

| Adres | Little endian | Big endian |
| --- | --- | --- |
| `0x1000` | `0x78` | `0x12` |
| `0x1001` | `0x56` | `0x34` |
| `0x1002` | `0x34` | `0x56` |
| `0x1003` | `0x12` | `0x78` |

x86-64 işlemciler her zaman little endian çalışır. Bayt sırasını ters çevirmek için `BSWAP` komutu bulunur. 16 bitlik değerler için `XCHG` veya `ROL` ile 8 bitlik döndürme kullanılabilir. Yeni işlemcilerde bellekten okurken sırayı çeviren `MOVBE` komutu da vardır.

## Kernel tarafında

Kernel'in endianness ile karşılaştığı yerler genellikle şunlardır:

- Ağ yığını: Ethernet, IPv4, UDP ve TCP başlıklarındaki çok baytlı alanlar.
- Dosya sistemi sürücüleri: diskteki yapıların okunması.
- Boot protokolleri ve firmware tabloları: ACPI ve UEFI yapıları little endian'dır.
- Aygıt sürücüleri: bazı aygıtlar big endian veri bekler.

## Implementasyon notları

Bayt sırasını çeviren küçük yardımcı fonksiyonlar yazmak işi kolaylaştırır. GCC ve Clang bunun için hazır builtin fonksiyonlar sunar:

```c
#include <stdint.h>

/* Ağdan gelen 16 bitlik değeri işlemcinin düzenine çevirir. */
static inline uint16_t ntohs(uint16_t x)
{
    return __builtin_bswap16(x);
}

static inline uint32_t ntohl(uint32_t x)
{
    return __builtin_bswap32(x);
}
```

Bu örnek, kernel'in little endian bir işlemcide çalıştığını varsayar. Birden fazla mimariyi hedefleyen bir kernel'de bu fonksiyonlar `__BYTE_ORDER__` makrosuna bakarak yazılmalıdır.

Bir yapıyı diskten okuyup doğrudan `struct` üzerine kopyalamak yaygın bir yöntemdir. Bu yöntem yalnızca diskteki düzen ile işlemcinin düzeni aynıysa doğru sonuç verir.

## Mimariye göre farklar

| Mimari | Davranış |
| --- | --- |
| x86-64 | Her zaman little endian |
| ARM64 | Veri erişimi için iki düzen de desteklenebilir, `SCTLR_EL1.EE` biti belirler. Pratikte neredeyse her zaman little endian kullanılır. Komutlar her zaman little endian'dır. |
| RISC-V | Temel olarak little endian. Privileged spesifikasyonu veri erişimi için isteğe bağlı big endian desteği tanımlar. Komutlar her zaman little endian'dır. |

## Sık yapılan hatalar

- Ağ başlığındaki port veya adres alanlarını çevirmeden karşılaştırmak.
- Hem okurken hem yazarken çevirmeyi unutmak. Okumada çevrilen bir alan, geri yazılırken de çevrilmelidir.
- Tek baytlık alanları da çevirmeye çalışmak. Tek bayt için endianness yoktur.
- Bit alanlarının (bit field) bellekteki yerleşiminin derleyiciye bağlı olduğunu unutmak.

## İlgili notlar

- MMIO ve port I/O: [mmio-ve-port-io.md](mmio-ve-port-io.md)
- ext2: [../08-filesystems/ext2.md](../08-filesystems/ext2.md)

## Kaynaklar

- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 1, Bölüm 1.3.1: Bit and Byte Order
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 2A ve 2B: BSWAP ve MOVBE komut tanımları
- Arm® Architecture Reference Manual for A-profile architecture, SCTLR_EL1 register tanımı (EE ve E0E bitleri)
- The RISC-V Instruction Set Manual, Volume I: Unprivileged ISA, "Memory" bölümü
- RFC 1700, "Assigned Numbers", Data Notations bölümü; <https://www.rfc-editor.org/rfc/rfc1700> (erişim: 1 Ekim 2026)
- GCC Manual, "Other Built-in Functions Provided by GCC" (`__builtin_bswap16`, `__builtin_bswap32`); <https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html> (erişim: 20 Ekim 2026)
