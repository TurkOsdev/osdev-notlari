# Söz dizimi farkları: Intel (NASM) ve AT&T (GAS)

x86 assembly iki farklı söz dizimiyle (syntax) yazılır: Intel ve AT&T. İkisi de aynı makine kodunu üretir, yalnızca yazılış biçimleri farklıdır. NASM Intel söz dizimini kullanır. GNU assembler (GAS) ise varsayılan olarak AT&T söz dizimini kullanır. OSDev kaynaklarında iki biçim de sık geçtiği için ikisini de okuyabilmek gerekir.

## Kapsam

Bu not x86-64 için iki söz dizimi arasındaki temel farkları anlatır.

## Problem

Bir öğreticide `mov rax, 1` yazarken başka birinde `movq $1, %rax` görülür. GCC'nin ürettiği assembly çıktısı ve `objdump` varsayılan olarak AT&T biçimindedir. Farkları bilmeden bu kodları birbirine çevirmek veya okumak hataya açıktır. En tehlikeli fark işlenen (operand) sırasıdır: yanlış okunursa kaynak ve hedef yer değiştirir.

## Temel farklar

| Konu | Intel (NASM) | AT&T (GAS) |
| --- | --- | --- |
| İşlenen sırası | `hedef, kaynak` | `kaynak, hedef` |
| Register | `rax` | `%rax` |
| Sabit değer | `5` | `$5` |
| Boyut | işlenenden anlaşılır veya `qword` yazılır | komutun sonuna harf eklenir: `b`, `w`, `l`, `q` |
| Bellek erişimi | `[rbx + rcx*8 + 16]` | `16(%rbx,%rcx,8)` |

Aynı işi yapan iki örnek:

```asm
; NASM (Intel)
mov  rax, 1
mov  qword [rbx + 8], 0
add  rax, [rsp + 16]
```

```asm
# GAS (AT&T)
movq $1, %rax
movq $0, 8(%rbx)
addq 16(%rsp), %rax
```

## Direktif farkları

Komutların dışında, assembler'a verilen direktifler de farklıdır:

| İş | NASM | GAS |
| --- | --- | --- |
| Bölüm seçmek | `section .text` | `.section .text` |
| Sembolü dışa açmak | `global _start` | `.globl _start` |
| Bayt tanımlamak | `db 0x41` | `.byte 0x41` |
| Boş alan ayırmak | `resb 64` | `.skip 64` |
| Yorum | `;` | `#` |

## Araçlarda söz dizimi seçmek

- GAS, dosyanın başına `.intel_syntax noprefix` yazılırsa Intel söz dizimini kabul eder.
- GCC'ye `-masm=intel` verilirse ürettiği assembly Intel biçiminde olur. Bu seçenek inline assembly'nin yorumlanmasını da değiştirir.
- `objdump -d -M intel` ile çıktı Intel biçiminde gösterilir.
- GDB'de `set disassembly-flavor intel` komutu aynı işi yapar.

## Sık yapılan hatalar

- AT&T kodunu okurken işlenen sırasını Intel gibi okumak.
- AT&T'de sabit değerin başına `$` koymamak. `movq 5, %rax` sabit 5'i değil, 5 adresindeki değeri okur.
- NASM'da bellek işleneninin boyutu belli değilken `qword` veya `byte` yazmamak. NASM bu durumda hata verir.
- Inline assembly'yi Intel biçiminde yazıp GCC'yi `-masm=intel` olmadan çalıştırmak.

## İlgili notlar

- [x86-64 register'ları](registerlar.md)
- [Stack ve stack frame](stack.md)

## Kaynaklar

- GNU Assembler Manual, "i386-Dependent Features", AT&T Syntax versus Intel Syntax; <https://sourceware.org/binutils/docs/as/i386_002dVariations.html> (erişim: 3 Ekim 2026)
- NASM Manual; <https://www.nasm.us/doc/> (erişim: 25 Ekim 2026)
- GCC Manual, "x86 Options" (`-masm`); <https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html> (erişim: 25 Ekim 2026)
