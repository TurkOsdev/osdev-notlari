# MMIO ve port I/O

İşlemcinin bir aygıtla konuşabilmesi için aygıtın register'larına ulaşması gerekir. Bunun iki yolu vardır. MMIO (Memory-Mapped I/O) yönteminde aygıt register'ları fiziksel adres alanında bir yere yerleştirilir ve normal bellek erişimiyle okunur. Port I/O yönteminde ise aygıtlar ayrı bir adres alanında durur ve bu alana yalnızca özel komutlarla erişilir.

## Kapsam

Port I/O yalnızca x86 ve x86-64'te vardır. Bu not x86-64'ü esas alır, ARM64 ve RISC-V için kısa bir karşılaştırma verir.

## Problem

İşlemci, aygıtın içinde ne olduğunu bilmez. Bir seri port'a karakter göndermek, bir disk denetleyicisine komut vermek veya ekran kartının ayarını değiştirmek için hepsinde aynı şey gerekir: aygıtın belirli bir register'ına bir değer yazmak ya da oradan bir değer okumak. Bu register'ların bir adresi olmalıdır.

## Donanım seviyesinde

### Port I/O

x86'da bellekten ayrı, 16 bitlik bir I/O adres alanı vardır. Bu alanda 65536 port bulunur. Erişim `IN` ve `OUT` komutlarıyla yapılır. Örneğin COM1 seri port'u `0x3F8` adresinden başlar, eski PIC ise `0x20` ve `0x21` port'larını kullanır.

Port numarası 255'ten küçükse doğrudan komut içinde yazılabilir. Daha büyük port numaraları `DX` register'ı üzerinden verilir. Okunan veya yazılan değer `AL`, `AX` ya da `EAX` register'ında durur.

C kodundan bu komutlara inline assembly ile ulaşılır:

```c
#include <stdint.h>

static inline void outb(uint16_t port, uint8_t val)
{
    __asm__ volatile ("outb %0, %1" : : "a"(val), "Nd"(port));
}

static inline uint8_t inb(uint16_t port)
{
    uint8_t ret;
    __asm__ volatile ("inb %1, %0" : "=a"(ret) : "Nd"(port));
    return ret;
}
```

Kullanıcı modunda bu komutlar varsayılan olarak çalışmaz. İzin, RFLAGS içindeki IOPL alanına ve TSS (Task State Segment) içindeki I/O izin bitmap'ine bağlıdır.

### MMIO

MMIO'da aygıt register'ları fiziksel adres alanının bir parçası gibi görünür. O adrese yapılan okuma veya yazma RAM'e gitmez, aygıta gider. Erişim için `MOV` gibi sıradan komutlar yeterlidir. Modern aygıtların çoğu (APIC, PCIe aygıtları, framebuffer) MMIO kullanır.

## Kernel tarafında

MMIO adresleri fiziksel adreslerdir. Paging açıkken kernel'in bu adresleri sanal adres alanına eşlemesi gerekir. Eşleme yapılırken sayfa önbelleğe alınmamalıdır (uncacheable). Aksi halde işlemci yazılan değeri önbellekte tutabilir ve aygıta hiç göndermeyebilir. x86-64'te bu, sayfa tablosu girdisindeki PCD ve PWT bayrakları veya PAT ile ayarlanır.

C tarafında MMIO adresine erişen işaretçi `volatile` olmalıdır. `volatile` olmayan bir erişimi derleyici silebilir, birleştirebilir veya sırasını değiştirebilir.

```c
/* Aygıt register'ını oku. Adresin önceden eşlendiği varsayılır. */
static inline uint32_t mmio_read32(uintptr_t addr)
{
    return *(volatile uint32_t *)addr;
}

static inline void mmio_write32(uintptr_t addr, uint32_t val)
{
    *(volatile uint32_t *)addr = val;
}
```

## Mimariye göre farklar

| Mimari | Davranış |
| --- | --- |
| x86-64 | Hem port I/O hem MMIO var |
| ARM64 | Yalnızca MMIO. Aygıt belleği MAIR register'ında Device türü olarak işaretlenir. |
| RISC-V | Yalnızca MMIO. Bellek türünü platform belirler, Svpbmt eklentisi sayfa bazında ayar sağlar. |

## Sık yapılan hatalar

- MMIO işaretçisini `volatile` yapmamak. Kod derleyici optimizasyonu açıkken çalışmayı bırakır.
- MMIO bölgesini normal, önbelleğe alınabilir bellek olarak eşlemek.
- Aygıtın beklediği erişim genişliğine uymamak. 32 bitlik bir register'a 8 bitlik yazma her aygıtta çalışmaz.
- Fiziksel MMIO adresini paging açıldıktan sonra eşlemeden kullanmak.

## İlgili notlar

- Endianness: [endianness.md](endianness.md)
- Seri port (UART): [../07-drivers/uart.md](../07-drivers/uart.md)

## Kaynaklar

- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 1, "Input/Output" bölümü
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 3A, "Memory Cache Control" bölümü (önbellek türleri, PAT)
- Arm® Architecture Reference Manual for A-profile architecture, "Memory types and attributes" bölümü
- The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Svpbmt eklentisi
- GCC Manual, "Extended Asm"; <https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html> (erişim: 9 Ekim 2026)
