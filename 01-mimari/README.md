# 01 — Mimari

İşletim sisteminin üzerinde çalıştığı donanımın modeli: komut seti mimarileri, çalışma modları, ayrıcalık seviyeleri ve bellek hiyerarşisi.

Kernel'in yaptığı işlerin çoğu doğrudan mimarinin dayattığı kurallardan doğar. Bu bölüm, sonraki bölümlerde tekrar tekrar başvurulacak temel modeli kurar.

## Konular

| Konu | Dosya | Durum |
| --- | --- | --- |
| ISA nedir, mimari ile implementasyon farkı | `isa-nedir.md` | ⬜ |
| x86 (IA-32): gerçek mod ve korumalı mod | `x86.md` | ⬜ |
| x86-64: long mode ve register seti | `x86-64.md` | ⬜ |
| ARM64 (AArch64): exception level'lar | `arm64.md` | ⬜ |
| RISC-V: privilege mode'lar ve eklentiler | `riscv.md` | ⬜ |
| Ayrıcalık seviyeleri (ring'ler, EL'ler, mode'lar) | `ayricalik-seviyeleri.md` | ⬜ |
| Endianness | [endianness.md](endianness.md) | ✅ |
| Bellek hiyerarşisi: cache ve TLB | `cache-ve-tlb.md` | ⬜ |
| Bellek modeli ve sıralama (memory ordering) | `bellek-modeli.md` | ⬜ |
| MMIO ve port I/O | [mmio-ve-port-io.md](mmio-ve-port-io.md) | ✅ |
| Mimari manual'ları nasıl okunur | `manual-okuma.md` | ⬜ |

---

**Durum işaretleri:** ⬜ yazılmadı · 🟡 yazılıyor · ✅ yayında

Bu bölüme katkıda bulunmadan önce [CONTRIBUTING.md](../CONTRIBUTING.md) ve [SABLON.md](../SABLON.md) dosyalarını okuyun. Listede olmayan bir konu önermek için önce issue açın.
