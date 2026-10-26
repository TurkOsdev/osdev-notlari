# Stack ve stack frame

Stack, fonksiyon çağrılarının dönüş adreslerini, yerel değişkenleri ve geçici olarak saklanan register'ları tutan bellek bölgesidir. Son giren ilk çıkar mantığıyla çalışır. Bir fonksiyon çağrıldığında stack'e yeni bir alan eklenir, fonksiyon döndüğünde bu alan geri bırakılır. Her fonksiyonun stack üzerinde kullandığı bu alana stack frame denir.

## Ön koşullar

- [x86-64 register'ları](registerlar.md)

## Kapsam

Bu not x86-64 mimarisini ve System V AMD64 ABI'sini esas alır. Assembly örnekleri NASM söz dizimiyle yazılmıştır.

## Problem

Bir fonksiyon başka bir fonksiyonu çağırdığında, çağrılan fonksiyonun işi bitince nereye döneceğini bilmesi gerekir. Fonksiyonun yerel değişkenleri için de bir yer gerekir. Bu yer sabit olamaz, çünkü aynı fonksiyon kendini tekrar çağırabilir (recursion) ve her çağrının kendi değişkenleri olmalıdır. Stack bu iki ihtiyacı da karşılar.

## Donanım seviyesinde

x86-64'te stack aşağı doğru büyür, yani yeni eklenen veri daha düşük adrese yazılır. `RSP` register'ı stack'in tepesini, yani en son yazılan değeri gösterir.

- `PUSH`: `RSP`'den 8 çıkarır, sonra değeri `RSP`'nin gösterdiği adrese yazar.
- `POP`: `RSP`'nin gösterdiği değeri okur, sonra `RSP`'ye 8 ekler.
- `CALL`: dönüş adresini stack'e push eder ve fonksiyona atlar.
- `RET`: stack'ten dönüş adresini pop eder ve oraya atlar.

Interrupt ve exception geldiğinde de işlemci stack'e veri yazar. Bu yüzden kernel'de stack'in her an geçerli bir bellek bölgesini göstermesi gerekir.

## Stack frame

Klasik bir fonksiyon başlangıcı ve bitişi şöyle görünür:

```asm
; NASM söz dizimi
fonksiyon:
    push rbp            ; çağıranın RBP değerini sakla
    mov  rbp, rsp       ; bu fonksiyonun frame başlangıcı
    sub  rsp, 32        ; yerel değişkenler için 32 bayt ayır

    ; ... fonksiyonun gövdesi ...

    mov  rsp, rbp       ; ayrılan alanı geri bırak
    pop  rbp            ; çağıranın RBP değerini geri yükle
    ret
```

Bu düzende her frame, bir önceki frame'in `RBP` değerini tutar. Böylece frame'ler bir zincir oluşturur. Hata ayıklayıcılar ve kernel'in kendi stack trace kodu bu zinciri takip ederek çağrı sırasını bulabilir.

Derleyici optimizasyon açıkken `RBP` kullanmayabilir. Zincirin korunması isteniyorsa GCC ve Clang'e `-fno-omit-frame-pointer` verilir.

## Kernel tarafında

Kernel, ilk komutundan itibaren kendi stack'ini kullanmalıdır. Bazı bootloader'lar bir stack verir, ama kernel genellikle kendi ayırdığı alana geçer:

```asm
; NASM söz dizimi
section .bss
align 16
stack_alt:
    resb 16384          ; 16 KiB stack
stack_ust:

section .text
global _start
_start:
    mov  rsp, stack_ust ; stack aşağı büyüdüğü için üst adresten başlanır
    call kernel_main
.dur:
    hlt
    jmp  .dur
```

İleride her thread için ayrı bir kernel stack'i gerekir. Ayrıca bazı exception'lar için ayrı stack kullanmak gerekir. Bu konu TSS ve IST ile ilgilidir: `04-kernel/tss.md`.

## Implementasyon notları

- System V ABI'ye göre `CALL` komutundan hemen önce `RSP` 16'nın katı olmalıdır. Bu kurala uyulmazsa SSE komutları kullanan kod hata verebilir.
- System V ABI, `RSP`'nin altındaki 128 baytı red zone olarak tanımlar. Fonksiyonlar bu alanı `RSP`'yi değiştirmeden kullanabilir. Kernel'de interrupt geldiğinde işlemci bu alanın üzerine yazar. Bu yüzden kernel kodu `-mno-red-zone` ile derlenir.
- Stack'in altına haritalanmamış bir sayfa (guard page) koymak, taşmayı page fault olarak yakalamayı sağlar.

## Sık yapılan hatalar

- `RSP`'yi ayrılan alanın alt adresine kurmak. Stack aşağı büyüdüğü için üst adres kullanılmalıdır.
- Kernel'i red zone açıkken derlemek. Hata rastgele bozulan yerel değişkenler olarak görülür.
- Stack'i çok küçük tutmak. Taşma, yakındaki başka verinin üzerine yazar ve hata çok uzakta ortaya çıkar.
- `PUSH` ve `POP` sayısını eşit tutmamak. `RET` yanlış adrese döner.

## İlgili notlar

- [x86-64 register'ları](registerlar.md)
- [IDT](../04-kernel/idt.md)

## Kaynaklar

- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 1, Bölüm 6: Procedure Calls, Interrupts, and Exceptions
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 2A ve 2B: PUSH, POP, CALL, RET komut tanımları
- System V Application Binary Interface, AMD64 Architecture Processor Supplement, Bölüm 3.2.2: The Stack Frame
- GCC Manual, "x86 Options" (`-mno-red-zone`); <https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html> (erişim: 23 Ekim 2026)
