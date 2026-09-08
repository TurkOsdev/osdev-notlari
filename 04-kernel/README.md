# 04 — Kernel

Kernel'in çekirdek yapıları: tanımlayıcı tablolar, interrupt ve exception yönetimi, kesme denetleyicileri, zamanlayıcılar ve system call mekanizmaları.

Bu bölüm, kernel'in donanımla kurduğu sözleşmeyi anlatır. Bellek yönetimi ve process yönetimi ayrı bölümlerde ele alınır.

## Konular

| Konu | Dosya | Durum |
| --- | --- | --- |
| Kernel nedir, ne yapar | [kernel-nedir.md](kernel-nedir.md) | ✅ |
| Kernel mimarileri: monolitik, mikrokernel, hibrit | `kernel-mimarileri.md` | ✅ |
| Freestanding C ortamı ve kısıtları | `freestanding-c.md` | ⬜ |
| Kernel'in bellek yerleşimi ve higher half | `bellek-yerlesimi.md` | ⬜ |
| GDT (Global Descriptor Table) | `gdt.md` | ⬜ |
| Segmentation ve x86-64'te durumu | `segmentation.md` | ⬜ |
| IDT (Interrupt Descriptor Table) | `idt.md` | ⬜ |
| Exception'lar ve fault türleri | `exceptionlar.md` | ⬜ |
| TSS ve kernel stack'leri (IST) | `tss.md` | ⬜ |
| PIC (8259) | `pic.md` | ⬜ |
| APIC ve IOAPIC | `apic.md` | ⬜ |
| Timer'lar: PIT, HPET, APIC timer, TSC | `timerlar.md` | ⬜ |
| System call mekanizmaları (int 0x80, syscall/sysret) | `system-calllar.md` | ⬜ |
| Senkronizasyon: spinlock, mutex, atomik işlemler | `senkronizasyon.md` | ⬜ |
| SMP ve çekirdeklerin başlatılması | `smp.md` | ⬜ |
| Kernel panic, log ve hata ayıklama altyapısı | `panic-ve-log.md` | ⬜ |

---

**Durum işaretleri:** ⬜ yazılmadı · 🟡 yazılıyor · ✅ yayında

Bu bölüme katkıda bulunmadan önce [CONTRIBUTING.md](../CONTRIBUTING.md) ve [SABLON.md](../SABLON.md) dosyalarını okuyun. Listede olmayan bir konu önermek için önce issue açın.
