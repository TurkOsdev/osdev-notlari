# VGA metin modu

VGA metin modu, ekranı karakterlerden oluşan bir ızgara olarak kullanan eski bir görüntü modudur. Standart boyut 80 sütun ve 25 satırdır. Ekrana yazı yazmak için belirli bir bellek adresine karakter ve renk değerleri yazmak yeterlidir. BIOS ile açılan bir makinede kernel'in ekrana ilk yazısını yazmasının en kısa yolu budur.

## Ön koşullar

- [MMIO ve port I/O](../01-mimari/mmio-ve-port-io.md)

## Kapsam

Bu not x86 PC'de BIOS ile açılan sistemleri anlatır. UEFI ile açılan sistemlerde bu mod genelde bulunmaz. Örnekler C ile yazılmıştır.

## Problem

Kernel'in ilk aşamalarında ekrana bir şey yazmak için font, piksel çizimi veya sürücü kodu yazmak zahmetlidir. VGA metin modunda bu işlerin hepsini donanım yapar. Kernel yalnızca hangi karakterin hangi renkte nerede duracağını söyler.

## Donanım seviyesinde

Metin modu belleği fiziksel `0xB8000` adresinden başlar. Ekrandaki her hücre 2 bayttır:

- Düşük bayt: karakterin kodu. Karakter seti code page 437'dir, ilk 128 karakter ASCII ile aynıdır.
- Yüksek bayt: renk baytı. Düşük 4 bit yazı rengi, sonraki 3 bit arka plan rengi, en yüksek bit varsayılan olarak yanıp sönme (blink) bitidir.

Hücrelerin sırası soldan sağa ve yukarıdan aşağıyadır. `(satir, sutun)` konumundaki hücrenin indeksi `satir * 80 + sutun` olur.

16 renk vardır. Sık kullanılanlar:

| Değer | Renk |
| --- | --- |
| 0 | Siyah |
| 1 | Mavi |
| 2 | Yeşil |
| 4 | Kırmızı |
| 7 | Açık gri |
| 15 | Beyaz |

## Kernel tarafında

```c
#include <stdint.h>

#define GENISLIK 80
#define YUKSEKLIK 25

static volatile uint16_t *const ekran = (volatile uint16_t *)0xB8000;

void ekrana_yaz(int satir, int sutun, char c, uint8_t renk)
{
    ekran[satir * GENISLIK + sutun] = (uint16_t)renk << 8 | (uint8_t)c;
}

void ekrani_temizle(void)
{
    for (int i = 0; i < GENISLIK * YUKSEKLIK; i++)
        ekran[i] = (uint16_t)0x07 << 8 | ' ';
}
```

Bu örnek, `0xB8000` adresinin kernel'in sanal adres alanında aynı adreste erişilebilir olduğunu varsayar. Higher half kernel'de bu adres önce eşlenmelidir.

### İmleç

Yanıp sönen imleç (cursor) ayrı ayarlanır. İmlecin konumu, `0x3D4` ve `0x3D5` port'ları üzerinden VGA'nın CRT denetleyicisine yazılır:

```c
void imlec_tasi(int satir, int sutun)
{
    uint16_t konum = satir * GENISLIK + sutun;

    outb(0x3D4, 0x0F);
    outb(0x3D5, konum & 0xFF);
    outb(0x3D4, 0x0E);
    outb(0x3D5, (konum >> 8) & 0xFF);
}
```

`0x0F` konumun düşük baytını, `0x0E` yüksek baytını tutan register'dır.

## Implementasyon notları

- Bir terminal gibi davranmak için kernel, yazının o anki satır ve sütununu kendisi tutmalıdır. `\n` gibi karakterleri de kendisi işlemelidir.
- Ekran dolduğunda bütün satırlar bir yukarı kaydırılır ve son satır temizlenir.
- Bu mod modern sistemlerde giderek ortadan kalkıyor. UEFI ile açılan veya Limine gibi bir bootloader kullanan sistemlerde framebuffer kullanmak gerekir.

## Sık yapılan hatalar

- İşaretçiyi `volatile` yapmamak. Derleyici yazmaları silebilir.
- UEFI ile açılan bir makinede `0xB8000`'e yazıp ekranda bir şey görmeyi beklemek.
- Renk baytını unutmak. Renk 0 olursa siyah üzerine siyah yazılır ve hiçbir şey görünmez.

## İlgili notlar

- [MMIO ve port I/O](../01-mimari/mmio-ve-port-io.md)
- [Limine boot protokolü](../02-boot/limine.md)

## Kaynaklar

- FreeVGA Project, "Hardware Level VGA and SVGA Video Programming Information Page"; <http://www.osdever.net/FreeVGA/home.htm> (erişim: 9 Ekim 2026)
- FreeVGA Project, "CRT Controller Registers"; <http://www.osdever.net/FreeVGA/vga/crtcreg.htm> (erişim: 4 Ekim 2026)
- OSDev Wiki, "Printing To Screen"; <https://wiki.osdev.org/Printing_To_Screen> (erişim: 10 Ekim 2026)
- OSDev Wiki, "Text Mode Cursor"; <https://wiki.osdev.org/Text_Mode_Cursor> (erişim: 26 Ekim 2026)
