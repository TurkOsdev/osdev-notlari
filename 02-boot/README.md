# 02 — Boot

Bilgisayarın güç verilmesinden kernel'in ilk komutunun çalışmasına kadar geçen süreç: firmware, bootloader'lar ve boot protokolleri.

Bu bölüm, kernel'in hangi durumda (hangi mod, hangi bellek düzeni, hangi register değerleri ile) devraldığını açıklar.

## Konular

| Konu | Dosya | Durum |
| --- | --- | --- |
| Boot sürecine genel bakış | `boot-sureci.md` | ⬜ |
| BIOS ve legacy boot | `bios.md` | ⬜ |
| MBR ve boot sector | `mbr.md` | ⬜ |
| UEFI temelleri | `uefi.md` | ⬜ |
| UEFI boot services ve runtime services | `uefi-services.md` | ⬜ |
| GPT bölümleme ve EFI System Partition | `gpt-ve-esp.md` | ⬜ |
| Gerçek moddan korumalı ve long mode'a geçiş | `mod-gecisleri.md` | ⬜ |
| A20 hattı | `a20.md` | ⬜ |
| Multiboot ve Multiboot2 | `multiboot.md` | ⬜ |
| [Limine boot protokolü](./limine.md) | `limine.md` | 🟡 |
| GRUB yapılandırması | `grub.md` | ⬜ |
| Bootloader yazmak mı, hazır bootloader mı | `bootloader-secimi.md` | ⬜ |
| Secure Boot | `secure-boot.md` | ⬜ |
| Boot sırasında bellek haritasının alınması | `bellek-haritasi.md` | ⬜ |

---

**Durum işaretleri:** ⬜ yazılmadı · 🟡 yazılıyor · ✅ yayında

Bu bölüme katkıda bulunmadan önce [CONTRIBUTING.md](../CONTRIBUTING.md) ve [SABLON.md](../SABLON.md) dosyalarını okuyun. Listede olmayan bir konu önermek için önce issue açın.
