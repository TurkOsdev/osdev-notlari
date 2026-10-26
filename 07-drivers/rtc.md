# RTC ve CMOS

RTC (Real-Time Clock), bilgisayar kapalıyken bile saati ve tarihi tutan küçük bir saattir. Anakarttaki pil sayesinde çalışmaya devam eder. x86 PC'lerde RTC, CMOS denen küçük bir bellekle birlikte bulunur ve ikisine de aynı port'lar üzerinden erişilir. Kernel açılırken o anki tarihi ve saati RTC'den okur.

## Ön koşullar

- [MMIO ve port I/O](../01-mimari/mmio-ve-port-io.md)

## Kapsam

Bu not x86 PC'deki MC146818 uyumlu RTC'yi anlatır. Örnekler `outb` ve `inb` fonksiyonlarının tanımlı olduğunu varsayar.

## Problem

Timer'lar geçen süreyi ölçer ama şu anın tarihini bilmez. Makine her açıldığında sayaçlar sıfırdan başlar. Dosyalara tarih yazmak veya log kayıtlarına saat eklemek için gerçek zamanı veren bir kaynak gerekir.

## Donanım seviyesinde

CMOS'a iki port üzerinden erişilir:

- `0x70`: okunacak register'ın numarası buraya yazılır.
- `0x71`: seçilen register'ın değeri buradan okunur.

`0x70` port'unun 7. biti NMI (Non-Maskable Interrupt) girişini kapatır. Register seçerken bu bitin durumu da değişir, bu yüzden dikkatli kullanılmalıdır.

Zaman register'ları:

| Register | İçerik |
| --- | --- |
| `0x00` | Saniye |
| `0x02` | Dakika |
| `0x04` | Saat |
| `0x07` | Ayın günü |
| `0x08` | Ay |
| `0x09` | Yıl (son iki hane) |
| `0x0A` | Durum register'ı A |
| `0x0B` | Durum register'ı B |

Durum register'ı A'nın 7. biti, RTC'nin o anda değerleri güncellediğini gösterir. Bu bit 1 iken okunan değerler yarım kalmış olabilir.

Durum register'ı B'deki iki bit değerlerin biçimini belirler:

- Bit 2: 1 ise değerler ikili (binary), 0 ise BCD (Binary-Coded Decimal) biçimindedir.
- Bit 1: 1 ise saat 24 saatlik, 0 ise 12 saatlik biçimdedir.

BCD biçiminde her hane 4 bitte tutulur. Örneğin 45 saniye, `0x45` olarak okunur.

## Kernel tarafında

```c
static uint8_t cmos_oku(uint8_t reg)
{
    outb(0x70, reg);
    return inb(0x71);
}

static int guncelleniyor(void)
{
    return cmos_oku(0x0A) & 0x80;
}

static uint8_t bcd_cevir(uint8_t x)
{
    return (x & 0x0F) + (x >> 4) * 10;
}

void rtc_saat_oku(uint8_t *saat, uint8_t *dakika, uint8_t *saniye)
{
    while (guncelleniyor())
        ;

    *saniye = cmos_oku(0x00);
    *dakika = cmos_oku(0x02);
    *saat   = cmos_oku(0x04);

    if (!(cmos_oku(0x0B) & 0x04)) {
        *saniye = bcd_cevir(*saniye);
        *dakika = bcd_cevir(*dakika);
        *saat   = bcd_cevir(*saat & 0x7F);
    }
}
```

Bu örnekte 12 saatlik biçim ele alınmamıştır. 12 saatlik biçimde saat değerinin 7. biti öğleden sonrayı (PM) gösterir.

## Implementasyon notları

- Güncelleme bitini beklemek tek başına yeterli değildir. Bekleme bittikten hemen sonra güncelleme başlayabilir. Daha güvenli yöntem, değerleri iki kez okuyup iki okuma aynı olana kadar tekrarlamaktır.
- Yıl yalnızca son iki haneyle tutulur. Yüzyıl bilgisi bazı makinelerde ayrı bir CMOS register'ındadır. Bu register'ın numarası ACPI FADT tablosundaki `Century` alanından okunur.
- RTC'nin saati yerel saat de olabilir UTC de. Windows genelde yerel saat, Linux genelde UTC kullanır.
- RTC, IRQ 8 üzerinden periyodik interrupt da üretebilir. Bu kullanılırsa her interrupt'tan sonra durum register'ı C okunmalıdır. Okunmazsa bir sonraki interrupt gelmez.

## Mimariye göre farklar

| Mimari | Davranış |
| --- | --- |
| x86-64 | CMOS RTC, `0x70` ve `0x71` port'ları |
| ARM64 | Standart bir RTC yok. Platforma göre değişir, adres aygıt ağacından okunur. QEMU `virt` makinesinde PL031 kullanılır. |
| RISC-V | Standart bir RTC yok. Platforma göre değişir, adres aygıt ağacından okunur. |

UEFI ile açılan sistemlerde runtime services içindeki `GetTime()` fonksiyonu mimariden bağımsız bir yol sunar.

## Sık yapılan hatalar

- Değerleri BCD'den çevirmeden kullanmak. 45 saniye 69 olarak görünür.
- 12 saatlik biçimde PM bitini temizlemeden BCD çevirmesi yapmak.
- Periyodik interrupt kullanırken durum register'ı C'yi okumamak.

## İlgili notlar

- [PIC (8259)](../04-kernel/pic.md)

## Kaynaklar

- Motorola MC146818A Real-Time Clock Plus RAM veri sayfası
- ACPI Specification, FADT tablosu, `CENTURY` alanı; <https://uefi.org/specifications> (erişim: 8 Ekim 2026)
- UEFI Specification, "Time Services" (`GetTime()`); <https://uefi.org/specifications> (erişim: 17 Ekim 2026)
- OSDev Wiki, "CMOS"; <https://wiki.osdev.org/CMOS> (erişim: 4 Ekim 2026)
