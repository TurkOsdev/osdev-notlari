# Seri port (UART)

UART (Universal Asynchronous Receiver-Transmitter), verileri tek bir hat üzerinden bit bit gönderen ve alan bir aygıttır. PC'lerdeki seri port'lar (COM1, COM2) 16550 uyumlu bir UART kullanır. OSDev'de seri port'un en önemli kullanımı hata ayıklamadır: kernel daha ekrana yazı yazamazken bile seri port'a metin gönderebilir ve bu metin QEMU'da terminalde görülebilir.

## Ön koşullar

- [MMIO ve port I/O](../01-mimari/mmio-ve-port-io.md)

## Kapsam

Bu not x86 PC'deki 16550 UART'ı ve port I/O ile erişimi anlatır. Örnekler `outb` ve `inb` fonksiyonlarının tanımlı olduğunu varsayar.

## Problem

Kernel'in ilk aşamalarında ekran sürücüsü yoktur ya da henüz düzgün çalışmıyordur. Bir şeylerin ters gittiğini görmenin bir yolu gerekir. Seri port çok az kodla kurulur, interrupt veya bellek yönetimi gerektirmez ve çıktısı emülatörden kolayca alınır.

## Donanım seviyesinde

COM1'in başlangıç port'u `0x3F8`, COM2'nin ise `0x2F8`'dir. UART'ın register'ları bu başlangıç adresine göre sıralanır:

| Offset | Register |
| --- | --- |
| +0 | Veri register'ı (DLAB=1 iken bölenin düşük baytı) |
| +1 | Interrupt etkinleştirme (DLAB=1 iken bölenin yüksek baytı) |
| +2 | FIFO denetimi |
| +3 | Hat denetimi (LCR) |
| +4 | Modem denetimi (MCR) |
| +5 | Hat durumu (LSR) |
| +6 | Modem durumu |
| +7 | Boş register (scratch) |

DLAB (Divisor Latch Access Bit), LCR register'ının 7. bitidir. Bu bit 1 yapıldığında ilk iki register, hız ayarı için kullanılan bölen değerine dönüşür.

Hız, saniyedeki bit sayısıdır ve baud rate olarak geçer. UART'ın temel saati 115200'dür. Bölen değeri `115200 / baud rate` ile bulunur. Örneğin 38400 baud için bölen 3'tür.

## Kernel tarafında

Başlatma ve tek karakter gönderme şöyle yapılabilir:

```c
#define COM1 0x3F8

void seri_baslat(void)
{
    outb(COM1 + 1, 0x00);   /* interrupt'ları kapat */
    outb(COM1 + 3, 0x80);   /* DLAB'ı aç */
    outb(COM1 + 0, 0x03);   /* bölen düşük bayt: 3 (38400 baud) */
    outb(COM1 + 1, 0x00);   /* bölen yüksek bayt */
    outb(COM1 + 3, 0x03);   /* DLAB'ı kapat, 8 bit, parite yok, 1 stop biti */
    outb(COM1 + 2, 0xC7);   /* FIFO'yu aç ve temizle */
    outb(COM1 + 4, 0x03);   /* DTR ve RTS'yi aç */
}

void seri_yaz(char c)
{
    /* LSR'nin 5. biti: gönderme register'ı boş mu */
    while ((inb(COM1 + 5) & 0x20) == 0)
        ;
    outb(COM1, c);
}
```

Bu örnekte hata kontrolü yoktur ve aygıtın var olduğu varsayılır.

Gelen veriyi okumak için LSR'nin 0. bitine bakılır. Bu bit 1 ise veri register'ında okunacak bir bayt vardır.

QEMU'da seri port çıktısını terminalde görmek için `-serial stdio` seçeneği kullanılır.

## Implementasyon notları

- Gerçek donanımda seri port olmayabilir. Aygıtın olup olmadığı, MCR'deki loopback modu açılıp gönderilen bir baytın geri okunmasıyla sınanabilir.
- Satır sonu için yalnızca `\n` göndermek bazı terminallerde merdiven şeklinde çıktı verir. `\n` öncesinde `\r` göndermek bu sorunu çözer.
- Seri port'a gönderim yavaştır. Çok fazla log yazmak kernel'i belirgin şekilde yavaşlatabilir.
- Gelen veri için polling yerine IRQ 4 (COM1) kullanılabilir. Bunun için interrupt etkinleştirme register'ı ve MCR'deki OUT2 biti ayarlanır.

## Mimariye göre farklar

| Mimari | Davranış |
| --- | --- |
| x86-64 | 16550, port I/O ile `0x3F8` adresinde |
| ARM64 | Genelde PL011 UART, MMIO ile. QEMU `virt` makinesinde `0x09000000` adresinde. |
| RISC-V | QEMU `virt` makinesinde 16550 uyumlu UART, MMIO ile `0x10000000` adresinde |

ARM64 ve RISC-V'de UART'ın adresi sabit değildir. Doğru adres aygıt ağacından (device tree) veya ACPI tablolarından okunmalıdır.

## Sık yapılan hatalar

- LSR'yi kontrol etmeden ardı ardına yazmak. Karakterler kaybolur.
- DLAB'ı açık bırakmak. Veri register'ına yazılan baytlar bölene gider ve hiçbir şey gönderilmez.
- QEMU'yu `-serial stdio` olmadan çalıştırıp çıktı aramak.

## İlgili notlar

- [PIC (8259)](../04-kernel/pic.md)
- [QEMU ile emülasyon](../00-giris/qemu.md)

## Kaynaklar

- Texas Instruments, "PC16550D Universal Asynchronous Receiver/Transmitter with FIFOs" veri sayfası
- OSDev Wiki, "Serial Ports"; <https://wiki.osdev.org/Serial_Ports> (erişim: 24 Ekim 2026)
- QEMU dokümantasyonu, "Invocation" (`-serial`); <https://www.qemu.org/docs/master/system/invocation.html> (erişim: 9 Ekim 2026)
