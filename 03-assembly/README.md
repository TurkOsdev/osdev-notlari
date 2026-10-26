# 03 — Assembly

Kernel geliştirmede assembly'nin kaçınılmaz olduğu yerler: mod geçişleri, interrupt giriş noktaları, context switch ve C ile assembly arasındaki sınır.

Amaç assembly'yi baştan sona öğretmek değil, kernel yazarken ihtiyaç duyulan kısmı ve C ile nasıl birlikte çalıştığını açıklamaktır.

## Konular

| Konu | Dosya | Durum |
| --- | --- | --- |
| Assembly'e giriş: kernel'de nerede gerekir | `giris.md` | ⬜ |
| Söz dizimi farkları: Intel (NASM) ve AT&T (GAS) | [sozdizimi.md](sozdizimi.md) | ✅ |
| x86-64 register'ları | [registerlar.md](registerlar.md) | ✅ |
| Adresleme modları | `adresleme-modlari.md` | ⬜ |
| Sık kullanılan komutlar | `komutlar.md` | ⬜ |
| Stack ve stack frame | [stack.md](stack.md) | ✅ |
| Calling convention (System V AMD64) | `calling-convention.md` | ⬜ |
| Ayrıcalıklı komutlar ve kontrol register'ları | `ayricalikli-komutlar.md` | ⬜ |
| Interrupt ve exception giriş noktaları | `interrupt-girisleri.md` | ⬜ |
| GCC / Clang inline assembly | `inline-assembly.md` | ⬜ |
| Assembler ve linker'ın işleyişi | `assembler-ve-linker.md` | ⬜ |
| Linker script'leri | `linker-script.md` | ⬜ |

---

**Durum işaretleri:** ⬜ yazılmadı · 🟡 yazılıyor · ✅ yayında

Bu bölüme katkıda bulunmadan önce [CONTRIBUTING.md](../CONTRIBUTING.md) ve [SABLON.md](../SABLON.md) dosyalarını okuyun. Listede olmayan bir konu önermek için önce issue açın.
