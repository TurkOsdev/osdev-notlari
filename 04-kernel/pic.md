# PIC (8259)

PIC (Programmable Interrupt Controller), aygıtlardan gelen interrupt isteklerini toplayıp işlemciye ileten çiptir. x86 bilgisayarlarda Intel 8259A uyumlu iki PIC bulunur. Klavye, timer ve seri port gibi aygıtlar interrupt isteğini doğrudan işlemciye değil, PIC'e gönderir. PIC bu istekleri sıraya koyar ve işlemciye hangi vektörün çalışacağını bildirir.

Modern sistemlerde PIC'in yerini APIC almıştır. Yine de PIC her x86 makinede uyumluluk için bulunur ve kernel açıldığında çoğu zaman ilk karşılaşılan interrupt denetleyicisidir.

## Ön koşullar

- [IDT](idt.md)
- [MMIO ve port I/O](../01-mimari/mmio-ve-port-io.md)

## Kapsam

Bu not x86 ve x86-64'e özgüdür. Örnekler C ile yazılmıştır ve `outb`, `inb` fonksiyonlarının tanımlı olduğunu varsayar.

## Problem

Eski x86 işlemcilerde aygıt interrupt'ları için tek bir giriş vardır: INTR hattı. Oysa aynı makinede birçok aygıt interrupt üretir. Bir aracı gerekir: hangi aygıtın istek yaptığını bilen, aynı anda gelen istekleri önceliğe göre sıralayan ve istenmeyen aygıtları susturabilen bir aracı. PIC bu işi yapar.

## Donanım seviyesinde

Her PIC'in 8 giriş hattı vardır. Bu hatlara IRQ (Interrupt Request) denir. İki PIC zincir (cascade) şeklinde bağlıdır:

- Master PIC: IRQ 0-7. Komut port'u `0x20`, veri port'u `0x21`.
- Slave PIC: IRQ 8-15. Komut port'u `0xA0`, veri port'u `0xA1`.
- Slave PIC, master'ın IRQ 2 hattına bağlıdır. Bu yüzden IRQ 2 bir aygıta ait değildir.

Sık görülen IRQ'lar:

| IRQ | Aygıt |
| --- | --- |
| 0 | PIT timer |
| 1 | PS/2 klavye |
| 4 | COM1 seri port |
| 8 | RTC |
| 12 | PS/2 fare |
| 14 | Birincil ATA |

Bir IRQ geldiğinde PIC, o IRQ'ya karşılık gelen vektör numarasını işlemciye verir. Vektör numarası, IRQ numarasına PIC'in ayarlanmış başlangıç değeri (offset) eklenerek bulunur.

## Kernel tarafında

### Yeniden eşleme (remap)

BIOS, master PIC'i 8. vektörden başlatır. Yani IRQ 0-7, vektör 8-15'e düşer. Ancak 0-31 arası vektörler işlemcinin exception'ları için ayrılmıştır. Örneğin vektör 8 double fault'tur. Timer her tetiklendiğinde kernel bunu double fault sanar.

Bu yüzden kernel PIC'i başlangıçta yeniden ayarlar. Ayar, ICW (Initialization Command Word) denen dört baytlık bir diziyle yapılır:

```c
#define PIC1_KOMUT 0x20
#define PIC1_VERI  0x21
#define PIC2_KOMUT 0xA0
#define PIC2_VERI  0xA1

void pic_yeniden_esle(void)
{
    /* ICW1: başlatma, ICW4 de gönderilecek */
    outb(PIC1_KOMUT, 0x11);
    outb(PIC2_KOMUT, 0x11);

    /* ICW2: vektör başlangıçları */
    outb(PIC1_VERI, 0x20);  /* IRQ 0-7  -> vektör 32-39 */
    outb(PIC2_VERI, 0x28);  /* IRQ 8-15 -> vektör 40-47 */

    /* ICW3: zincir bilgisi */
    outb(PIC1_VERI, 0x04);  /* slave, IRQ 2 hattında (bit 2) */
    outb(PIC2_VERI, 0x02);  /* slave'in kimliği 2 */

    /* ICW4: 8086 modu */
    outb(PIC1_VERI, 0x01);
    outb(PIC2_VERI, 0x01);

    /* Başlangıçta bütün IRQ'ları maskele */
    outb(PIC1_VERI, 0xFF);
    outb(PIC2_VERI, 0xFF);
}
```

Bazı eski donanımlar komutlar arasında kısa bir bekleme ister. Bunun için genelde kullanılmayan `0x80` port'una bir bayt yazılır. QEMU'da bu bekleme gerekmez.

### Maskeleme

Veri port'una yazılan bayt, IMR (Interrupt Mask Register) değeridir. Bir bit 1 ise o IRQ maskelenmiştir ve işlemciye iletilmez. Örneğin yalnızca timer ve klavyeyi açmak için master'a `0xFC` yazılır.

### EOI

Handler işini bitirdiğinde PIC'e EOI (End Of Interrupt) göndermelidir. EOI gönderilmezse PIC aynı veya daha düşük öncelikli IRQ'ları bir daha iletmez.

```c
void pic_eoi(uint8_t irq)
{
    if (irq >= 8)
        outb(PIC2_KOMUT, 0x20);
    outb(PIC1_KOMUT, 0x20);
}
```

Slave'den gelen bir IRQ için EOI hem slave'e hem master'a gönderilir.

## Implementasyon notları

- APIC kullanılacaksa PIC devre dışı bırakılmalıdır. Bunun için önce yeniden eşleme yapılır, sonra iki veri port'una da `0xFF` yazılır. Yeniden eşleme, maskeleme öncesinde gelebilecek bir sahte (spurious) interrupt'ın exception vektörüne düşmesini engeller.
- IRQ 7 ve IRQ 15 sahte interrupt olarak gelebilir. Bunun gerçek olup olmadığı ISR (In-Service Register) okunarak anlaşılır. Sahte bir IRQ 7 için EOI gönderilmez. Sahte bir IRQ 15 için yalnızca master'a EOI gönderilir.

## Mimariye göre farklar

| Mimari | Davranış |
| --- | --- |
| x86-64 | 8259 PIC uyumluluk için var, yerini APIC ve IOAPIC almıştır |
| ARM64 | PIC yok, GIC (Generic Interrupt Controller) kullanılır |
| RISC-V | PIC yok, PLIC veya AIA kullanılır |

## Sık yapılan hatalar

- PIC'i yeniden eşlemeden interrupt'ları açmak. Timer double fault gibi görünür.
- EOI göndermeyi unutmak. İlk interrupt gelir, sonrakiler hiç gelmez.
- Slave IRQ'larında yalnızca slave'e EOI göndermek.
- IRQ 2'yi maskelemek. Bu durumda slave PIC'e bağlı hiçbir IRQ gelmez.

## İlgili notlar

- [IDT](idt.md)
- [RTC ve CMOS](../07-drivers/rtc.md)
- [Seri port (UART)](../07-drivers/uart.md)

## Kaynaklar

- Intel 8259A Programmable Interrupt Controller (8259A/8259A-2) veri sayfası
- OSDev Wiki, "8259 PIC"; <https://wiki.osdev.org/8259_PIC> (erişim: 7 Ekim 2026)
