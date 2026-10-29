# 07 — Drivers

Donanımla konuşma: veri yolları, cihaz keşfi, kesme tabanlı sürücüler ve yaygın aygıt sınıfları.

Sürücüler, kernel kodunun büyük bölümünü oluşturur. Bu bölüm önce genel erişim yöntemlerini, sonra tek tek aygıtları ele alır.

## Konular

| Konu | Dosya | Durum |
| --- | --- | --- |
| Sürücü nedir, kernel içinde nerede durur | [surucu-nedir.md](surucu-nedir.md) | ✅ |
| Donanım erişimi: port I/O ve MMIO | [donanim-erisimi.md](donanim-erisimi.md) | ✅ |
| Kesme tabanlı sürücüler ve interrupt handler tasarımı | [kesme-tabanli-suruculer.md](kesme-tabanli-suruculer.md) | ✅ |
| DMA | `dma.md` | ⬜ |
| ACPI ve tabloları | `acpi.md` | ⬜ |
| PCI ve PCIe cihaz keşfi | `pci.md` | ⬜ |
| Seri port (UART) | [uart.md](uart.md) | ✅ |
| PS/2 klavye ve fare | [ps2.md](ps2.md) | ✅ |
| RTC ve CMOS | [rtc.md](rtc.md) | ✅ |
| ATA/IDE ve AHCI | `ata-ve-ahci.md` | ⬜ |
| NVMe | `nvme.md` | ⬜ |
| USB mimarisi ve XHCI | `usb.md` | ⬜ |
| Ağ kartları (E1000, RTL8139) | `ag-kartlari.md` | ⬜ |
| Ses aygıtları (AC'97, Intel HDA) | `ses.md` | ⬜ |

---

**Durum işaretleri:** ⬜ yazılmadı · 🟡 yazılıyor · ✅ yayında

Bu bölüme katkıda bulunmadan önce [CONTRIBUTING.md](../CONTRIBUTING.md) ve [SABLON.md](../SABLON.md) dosyalarını okuyun. Listede olmayan bir konu önermek için önce issue açın.
