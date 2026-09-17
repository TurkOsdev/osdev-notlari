# IDT (Interrupt Descriptor Table)

IDT (Interrupt Descriptor Table), işlemcinin bir interrupt ya da exception oluştuğunda hangi koda atlayacağını bulduğu tablodur. Kernel bu tabloyu bellekte kurar, adresini IDTR register'ına yazar ve o andan itibaren akışı bölen her olay — timer'ın dolması, klavyeye basılması, sıfıra bölme, haritalanmamış bir adrese erişim — kernel'in önceden belirlediği bir adreste yürütmeye dönüşür. Bu tablo kurulmadan işlemcinin yapabileceği tek şey vardır: hatayı büyütmek ve makineyi yeniden başlatmak.

Tablonun kendisi karmaşık değildir; en fazla 256 girdiden oluşur ve her girdi esas olarak bir adres ile birkaç bayrak taşır. Zorluk biçiminde değil, girdinin gösterdiği adrese ulaşana kadar işlemcinin verdiği kararlarda ve o adrese varıldığında kernel'in devraldığı durumdadır. Gate'in taşıdığı segment seçicisi, ayrıcalık seviyesi ve stack indeksi, "nereye atlanacağı" sorusunun yanında "hangi yetkiyle ve hangi stack'te çalışılacağı" sorusunu da yanıtlar. Bu not IDT girdisinin alanlarını, girişte donanımın yaptığı işi ve kernel'in bu mekanizmanın üzerine kurması gereken giriş kodunu anlatır.

## Ön koşullar

- [Kernel nedir](kernel-nedir.md) — kernel moduna giriş yolları
- [x86-64 register'ları](../03-assembly/registerlar.md) — RFLAGS, segment register'ları ve MSR'lar
- GDT ve segment seçicileri: `04-kernel/gdt.md`

## Kapsam

Bu not x86-64 mimarisini ve long mode'u esas alır; 32-bit protected mode farkları yalnızca gerektiği yerde belirtilir. Assembly örnekleri NASM (Intel) söz dizimiyle yazılmıştır.

IDT, vektör numarasını handler adresine çeviren tabloyla sınırlıdır. Vektör numarasının nereden geldiği (PIC ve APIC) `04-kernel/pic.md` ve `04-kernel/apic.md`, tek tek exception'ların anlamı ve hata kodlarının biçimi `04-kernel/exceptionlar.md`, IST'nin dayandığı TSS yapısı `04-kernel/tss.md` notlarının konusudur.

## Problem

Çalışan bir programın akışını, programın haberi olmadan ve işbirliği olmadan başka bir yere aktarmak gerekir. Diskin işi bittiğinde, timer dolduğunda ya da program geçersiz bir adres kullandığında işlemcinin bir yere atlaması şarttır. Asıl soru bu adresin nereden geleceğidir.

İki yaklaşım mümkündür. Birincisi tek bir sabit giriş noktası tanımlamaktır: ne olursa olsun aynı adrese atlanır, olayın ne olduğu ayrı bir register'dan okunur. İkincisi olayın türünü bir sayıya indirgeyip bu sayıyı tabloya indeks olarak kullanmaktır. x86 ikincisini seçti; olayın kimliği vektör numarasıdır ve doğru handler'a ulaşmak fazladan bir dallanma gerektirmez.

Ancak adres tek başına yetmez. Bu geçişin üç ek özelliği olmalıdır:

- **Hedef, kesilen programın belirlediği bir yer olamaz.** Adres kernel'in yazdığı, kullanıcı modunun değiştiremeyeceği bir tabloda durmalıdır.
- **Ayrıcalık seviyesi yükselmelidir.** Kullanıcı modunda oluşan bir page fault, kernel yetkisiyle çalışan bir kodda ele alınır. Bu yükselme kernel'in bir kararı değil, geçişin kendisinin parçasıdır.
- **Stack güvenilir olmalıdır.** Kullanıcı programının RSP'si geçersiz, hizalanmamış ya da kasıtlı olarak kernel belleğini gösteriyor olabilir. Kernel çerçevesinin bu stack'e yazılmaması gerekir.

IDT girdisinin yalnızca bir adres değil, bir segment seçicisi, bir DPL ve bir stack indeksi taşımasının nedeni budur. Girdi, bir fonksiyon işaretçisi değil, işlemciyle kernel arasındaki geçiş sözleşmesidir.

## Donanım seviyesinde

### IDTR ve tablonun konumu

Tablonun yeri IDTR register'ında tutulur. IDTR, 16 bitlik bir limit ile 64 bitlik bir taban adresinden oluşan 10 baytlık bir yapıdır ve `lidt` komutuyla yüklenir. Limit, tablonun **boyutundan bir eksiktir**: 256 girdilik tam bir tablo için 16 × 256 − 1 = 4095 yazılır.

Tablo bellekte herhangi bir yerde durabilir. Girdiler 16 bayt olduğundan taban adresinin 8 baytlık sınıra hizalanması önerilir; hizalanmamış bir taban her girdi okumasını iki önbellek satırına yayabilir. Tablonun 256 girdiden kısa olması da mümkündür, ancak tanımsız bir vektör geldiğinde sonuç sessiz bir yoksayma değil, yeni bir exception'dır.

`lidt` ayrıcalıklı bir komuttur. Buna karşılık `sidt` ayrıcalıklı değildir: CR4.UMIP açılmadıkça kullanıcı modundaki bir program IDT'nin adresini okuyabilir. Bu, kernel adreslerini sızdıran bilinen bir yoldur ve UMIP destekleniyorsa açılması gerekir.

### Gate descriptor

Long mode'da her IDT girdisi 16 bayttır; 32-bit protected mode'daki 8 baytlık girdinin adres alanı genişletilmiş ve IST alanı eklenmiş halidir.

| Bayt | Bit | Alan | Anlamı |
| --- | --- | --- | --- |
| 0–1 | 15:0 | offset (düşük) | handler adresinin 15:0 biti |
| 2–3 | — | segment seçici | GDT'deki 64 bit kernel kod segmenti |
| 4 | 2:0 | IST | 0 ise IST kullanılmaz, 1–7 ise TSS'teki ilgili stack |
| 4 | 7:3 | sıfır | ayrılmış |
| 5 | 3:0 | tip | 0xE interrupt gate, 0xF trap gate |
| 5 | 4 | sıfır | sistem descriptor'ı olduğunu belirtir |
| 5 | 6:5 | DPL | `int n` ile erişim için gereken en düşük ayrıcalık |
| 5 | 7 | P | girdi geçerli mi |
| 6–7 | 31:16 | offset (orta) | handler adresinin 31:16 biti |
| 8–11 | 63:32 | offset (yüksek) | handler adresinin 63:32 biti |
| 12–15 | — | sıfır | ayrılmış |

Long mode'da yalnızca iki gate tipi vardır. IA-32'deki task gate (tip 0x5) kaldırılmıştır; donanım destekli task switch 64 bit modda yoktur, dolayısıyla exception'ları ayrı bir TSS'e yönlendirme yöntemi de yoktur. Bu boşluğu IST doldurur.

Seçici alanı GDT'de 64 bit kod segmentini (L = 1) göstermelidir. İşlemci girişte CS'i bu seçiciden yükler ve yeni CPL, hedef segmentin DPL'i olur; kernel handler'ları için bu değer sıfırdır.

### Interrupt gate ile trap gate farkı

İki tip arasındaki tek fark RFLAGS.IF'e ne olduğudur:

| Tip | Girişte IF | Sonucu |
| --- | --- | --- |
| interrupt gate (0xE) | temizlenir | handler maskelenebilir interrupt'lara kapalı çalışır |
| trap gate (0xF) | değiştirilmez | handler çalışırken başka bir interrupt araya girebilir |

Aygıt interrupt'ları ve çoğu exception için interrupt gate kullanılır; handler'ın ilk satırlarında bölünmemesi, hem kendi veri yapılarını korur hem de `swapgs` gibi tek adımda tamamlanması gereken işleri güvenli kılar. Trap gate, kesilmesi sakıncası olmayan ya da uzun sürebilen girişler için anlamlıdır: hata ayıklama kesme noktası (#BP) ve `int 0x80` biçimindeki system call girişi tipik örneklerdir. Trap gate seçmek "interrupt'lar açık kalsın" demek değildir; yalnızca kararı kernel'e bırakır, handler istediği anda `cli` ile kapatabilir.

### Vektör alanı

0–31 arası vektörler mimari tarafından ayrılmıştır ve exception'lara karşılık gelir; 15 ile 22–31 arasındaki bir kısmı hâlâ rezervedir, ancak bu numaralar aygıt interrupt'ları için kullanılamaz. 32–255 arası kernel'in tasarrufundadır: kesme denetleyicisinin ürettiği vektörler, işlemciler arası interrupt'lar (IPI) ve varsa yazılım interrupt'ları buraya yerleştirilir.

Bazı exception'larda işlemci, çerçevenin üzerine ayrıca bir hata kodu iter. Bu ayrım handler'ın stack düzenini doğrudan etkilediğinden, hangi vektörün hata kodu ittiğini bilmek IDT kurulumunun zorunlu parçasıdır. Hata kodu iten vektörler 8 (#DF), 10 (#TS), 11 (#NP), 12 (#SS), 13 (#GP), 14 (#PF), 17 (#AC) ve 21 (#CP)'dir; AMD sanallaştırma eklentileri 29 (#VC) ve 30 (#SX)'u ekler. #DF ve #AC'nin hata kodu her zaman sıfırdır, yani bilgi taşımaz ama yine de itilir. Long mode'da hata kodu 8 bayt olarak itilir; anlamlı kısmı düşük 32 bittir.

### Girişte işlemcinin yaptığı iş

Bir vektör teslim edilirken sıra şudur:

1. Vektör numarası belirlenir. Exception'larda numara sabittir, harici interrupt'larda kesme denetleyicisi bildirir, yazılım interrupt'larında `int n` komutunun operandıdır.
2. IDT'den girdi okunur. Vektör limiti aşıyorsa #GP, girdinin P biti sıfırsa #NP üretilir; ikisinde de hata kodu hangi IDT girdisinin sorunlu olduğunu gösterir.
3. Yalnızca `int n`, `int3` ve `into` ile üretilen girişlerde DPL denetimi yapılır: CPL, gate'in DPL'inden büyükse #GP üretilir. Donanım interrupt'ları ve işlemcinin kendi ürettiği exception'lar bu denetimden geçmez.
4. CS ve RIP gate'ten yüklenir, CPL hedef segmentin DPL'ine göre belirlenir.
5. Stack seçilir. Gate'in IST alanı sıfır değilse stack işaretçisi koşulsuz olarak TSS'teki ilgili IST girdisinden alınır. IST sıfırsa ve ayrıcalık seviyesi yükseliyorsa TSS'teki RSP0 kullanılır. Ayrıcalık değişmiyorsa mevcut stack'te kalınır.
6. RSP, çerçeve itilmeden önce 16'nın katına hizalanır.
7. SS, RSP, RFLAGS, CS ve RIP itilir. Long mode'da SS:RSP ayrıcalık seviyesi değişmese bile **her zaman** itilir; 32-bit protected mode'da yalnızca seviye değiştiğinde itilirdi. Aynı akışa dönmek için iki ayrı dönüş düzeni tutmak gerekmemesinin nedeni budur.
8. Vektörün hata kodu varsa en son o itilir, yani çerçevenin en düşük adresinde durur.
9. RFLAGS'te TF, NT ve RF temizlenir; gate interrupt gate ise IF de temizlenir.

Handler'ın ilk komutu çalıştığında stack şöyle görünür:

```text
yüksek adres
  RSP+40   SS          kesilen akışın stack segmenti
  RSP+32   RSP         kesilen akışın stack işaretçisi
  RSP+24   RFLAGS      IF dahil eski bayraklar
  RSP+16   CS          düşük iki biti kesilen akışın CPL'i
  RSP+8    RIP         kesilen ya da hata veren komutun adresi
  RSP+0    hata kodu   yalnızca hata kodu iten vektörlerde
düşük adres
```

Hata kodu itilmeyen vektörlerde bu düzen sekiz bayt kayar: RSP doğrudan RIP'i gösterir. İtilen RIP'in hangi komutu gösterdiği exception'ın türüne bağlıdır; fault'larda hatalı komutun kendisi, trap'lerde bir sonraki komut itilir. Bu ayrım `04-kernel/exceptionlar.md` notunun konusudur.

Genel amaçlı register'ların hiçbiri kaydedilmez. Kesilen akışın RAX'i, RDI'si ve diğerleri hâlâ yerindedir ve handler'ın yazacağı ilk değerle kaybolur.

### Teslim edilemeyen vektörler

IDT'ye erişimin kendisi başarısız olabilir ve bu durumda üretilen yeni exception da teslim edilmek zorundadır. İşlemci ikinci exception'ı da teslim edemezse #DF (Double Fault, vektör 8) üretir. #DF'in handler'ı da yoksa ya da ona giderken üçüncü bir hata oluşursa işlemci triple fault durumuna girer; bu bir exception değildir, işlemcinin kapanma sinyalidir ve gerçek donanımda da emülatörde de makineyi yeniden başlatır.

Pratikte kernel geliştirirken görülen "hiçbir çıktı vermeden sürekli yeniden başlayan makine" tablosunun kaynağı neredeyse her zaman budur: IDT henüz kurulmamıştır ya da girdiler yanlıştır, ilk exception ikinciyi doğurur ve zincir triple fault'ta biter.

## Kernel tarafında

### Ne zaman kurulur

IDT, bootloader'ın bıraktığı durumun üzerine kurulan ilk yapılardan biridir. Sırası şöyledir: önce bir çıktı yolu (seri port ya da framebuffer), sonra kendi GDT'si, IST kullanılacaksa TSS, sonra IDT. Bellek allocator'ı, heap ve sürücüler bundan sonra gelir.

Bu sıra işlevsel bir zorunluluk değil, hata ayıklanabilirlik tercihidir. IDT kurulmadan önce yapılan bir hata ekranda iz bırakmadan makineyi yeniden başlatır; IDT kurulduktan sonra aynı hata vektör numarası, hata kodu ve RIP ile birlikte okunabilir bir çıktıya dönüşür. Aradaki fark, saatlerce süren bir aramayla tek satırlık bir düzeltme arasındaki farktır.

IDT kurulduktan sonra bile `sti` hemen çalıştırılmaz. Kesme denetleyicisi yapılandırılmadan interrupt'ları açmak, PIC'in varsayılan vektörleri exception vektörleriyle çakıştığı için klavyeye basıldığında #GP ya da anlamsız bir exception görülmesi demektir.

### Tablo ortak, IDTR işlemci başına

IDT genellikle tüm işlemciler için tek bir tablodur; her işlemcinin kendi kopyasını tutmasını gerektiren bir neden yoktur çünkü handler adresleri ortaktır. Ancak IDTR bir register'dır ve her işlemcide ayrı ayrı yüklenir: ikincil işlemciler başlatılırken kendi `lidt` komutlarını çalıştırır. IST alanının indeks olması bu paylaşımı mümkün kılar; aynı girdi her işlemcide o işlemcinin kendi TSS'indeki farklı stack'i gösterir.

### Girdinin yazılması

Girdi, alanları dağınık olduğu için elle kurulur:

```c
// 16 baytlık long mode gate descriptor'ı
struct idt_entry {
    uint16_t offset_low;   // handler adresinin 15:0 biti
    uint16_t selector;     // GDT'deki 64 bit kernel kod segmenti
    uint8_t  ist;          // 2:0 IST indeksi, kalan bitler sıfır
    uint8_t  type_attr;    // 3:0 tip, 4 sıfır, 6:5 DPL, 7 P
    uint16_t offset_mid;   // 31:16
    uint32_t offset_high;  // 63:32
    uint32_t reserved;     // sıfır
} __attribute__((packed));

// lidt'in beklediği 10 baytlık yapı
struct idtr {
    uint16_t limit;        // tablo boyutu - 1
    uint64_t base;
} __attribute__((packed));
```

`packed` niteleyicisi burada zorunludur: derleyici kendi hizalama kurallarıyla `ist` ile `type_attr` arasına dolgu koyabilir ve donanımın beklediği düzen bozulur.

```c
#define IDT_INTERRUPT_GATE 0x0E
#define IDT_TRAP_GATE      0x0F

static struct idt_entry idt[256] __attribute__((aligned(16)));

void idt_set_gate(uint8_t vector, void *handler,
                  uint8_t ist, uint8_t dpl, uint8_t type)
{
    uint64_t addr = (uint64_t)handler;

    idt[vector].offset_low  = addr & 0xFFFF;
    idt[vector].selector    = KERNEL_CS;   // GDT'deki kod segmenti seçicisi
    idt[vector].ist         = ist & 0x7;
    idt[vector].type_attr   = 0x80 | ((dpl & 0x3) << 5) | type;  // P = 1
    idt[vector].offset_mid  = (addr >> 16) & 0xFFFF;
    idt[vector].offset_high = addr >> 32;
    idt[vector].reserved    = 0;
}

void idt_load(void)
{
    static struct idtr idtr;

    idtr.limit = sizeof(idt) - 1;
    idtr.base  = (uint64_t)idt;
    __asm__ volatile ("lidt %0" :: "m"(idtr) : "memory");
}
```

`idtr` değişkeninin `static` olması bilinçlidir; `lidt` yalnızca yapının içeriğini okur, dolayısıyla yerel bir değişken de çalışır, ancak kernel'in erken aşamasında stack üzerindeki geçici yapılardan kaçınmak hata ayıklamayı kolaylaştırır.

### Giriş kodu neden assembly olmak zorunda

Gate'e doğrudan bir C fonksiyonunun adresi yazılamaz. Üç neden vardır: dönüşün `ret` ile değil `iretq` ile yapılması gerekir, hata kodu stack'te durduğu için C'nin gördüğü çerçeve düzeni bozulur ve C fonksiyonu kesilen akışın register'larını korumak zorunda olduğunu bilmez. Bu yüzden her vektör, kısa bir assembly giriş noktasına yönlendirilir; giriş noktası durumu düzene sokup C tarafını çağırır.

256 giriş noktasını elle yazmak yerine makro kullanılır. Giriş noktalarının tek işi, çerçeveyi bütün vektörlerde aynı biçime getirmek ve vektör numarasını kaydetmektir:

```asm
; NASM (Intel) söz dizimi

%macro ISR_NOERR 1              ; işlemcinin hata kodu itmediği vektörler
isr%1:
    push qword 0                ; hata kodunun yerine sahte değer
    push qword %1               ; vektör numarası
    jmp isr_common
%endmacro

%macro ISR_ERR 1                ; hata kodunu işlemcinin ittiği vektörler
isr%1:
    push qword %1               ; yalnızca vektör numarası
    jmp isr_common
%endmacro

isr_common:
    test byte [rsp + 24], 3     ; itilen CS'in düşük iki biti: eski CPL
    jz .kernelden               ; sıfırsa kesme zaten kernel modunda oluştu
    swapgs                      ; kullanıcıdan gelindiyse GS tabanını takas et
.kernelden:
    push rax
    push rcx
    ; ... kalan genel amaçlı register'lar aynı sırayla ...
    push r15

    cld                         ; çağrı kuralı DF = 0 varsayar
    mov rdi, rsp                ; ilk argüman: kaydedilmiş bağlamın adresi
    call interrupt_dispatch

    pop r15
    ; ... register'lar ters sırayla geri yüklenir ...
    pop rax

    test byte [rsp + 24], 3     ; dönüş kullanıcı moduna mı
    jz .kerneleDon
    swapgs
.kerneleDon:
    add rsp, 16                 ; vektör numarası ve hata kodu atılır
    iretq
```

Sahte hata kodunun amacı yalnızca düzen birliği değildir; hizalamayı da düzeltir, gerekçesi aşağıdaki implementasyon notlarındadır. Örnekte register kaydetme kısaltılmıştır, gerçek kodda RSP dışındaki on beş genel amaçlı register'ın tamamı kaydedilir.

### C tarafında dağıtım

`interrupt_dispatch`, kendisine gelen yapının içindeki vektör numarasına bakarak işi yönlendirir. Yaygın düzen, 256 elemanlı bir fonksiyon işaretçisi dizisi tutmak ve sürücülerin kendi vektörlerini bu diziye kaydetmesidir:

```c
struct interrupt_frame {
    uint64_t r15, r14, r13, r12, r11, r10, r9, r8;
    uint64_t rbp, rdi, rsi, rdx, rcx, rbx, rax;
    uint64_t vector;
    uint64_t error_code;
    uint64_t rip, cs, rflags, rsp, ss;   // işlemcinin ittiği çerçeve
};
```

Bu yapının alan sırası, assembly tarafındaki `push` sırasının tam tersidir ve ikisi birlikte değiştirilmek zorundadır. Aradaki uyumsuzluk derleme hatası vermez; yanlış register'ın ilk kullanıldığı yerde, çoğu zaman çok sonra ortaya çıkar.

Sürücülerin IDT'ye doğrudan dokunmaması, bu dizi üzerinden kayıt yapması tercih edilir. Çalışan bir sistemde IDT girdisini değiştirmek, girdinin yarısı yazılmışken o vektörün gelmesi ihtimali nedeniyle ayrıca dikkat gerektirir; C seviyesindeki dizide bir işaretçinin atomik olarak değiştirilmesi bu sorunu ortadan kaldırır.

### IST'nin kullanıldığı yerler

IST (Interrupt Stack Table), TSS'te tutulan yedi stack işaretçisidir ve gate'in IST alanı sıfırdan farklıysa işlemci koşulsuz olarak o stack'e geçer. Koşulsuzluk önemlidir: ayrıcalık seviyesi değişmese bile stack değişir. Bu, mevcut stack'e güvenilemeyen durumların tek çözümüdür.

Üç tipik kullanım vardır. #DF, çoğu zaman kernel stack'i taştığı için oluşur; taşmış bir stack'e çerçeve itmeye çalışmak triple fault demektir, dolayısıyla #DF'in kendi stack'i olmalıdır. NMI her an, kernel'in stack değiştirdiği kritik bölgelerin ortasında bile gelebilir. #MC (Machine Check) aynı şekilde makinenin tutarsız olduğu bir anda gelir.

IST'nin tuzağı, aynı IST girdisini kullanan bir exception'ın kendi içinde tekrar oluşmasıdır: işlemci stack işaretçisini yine aynı sabit değere yükler ve ikinci çerçeve birincinin üzerine yazılır. IST stack'leri iç içe girilebilir değildir. NMI handler'ının içinde bir page fault oluşması ve o fault'un `iret`'inin NMI bloklamasını kaldırması, bu sınıfın en bilinen örneğidir.

## Implementasyon notları

**Boş girdi bırakmayın.** Kurulumda 256 girdinin tamamı, hiçbir şey yapmasa bile vektör numarasını yazdırıp duran varsayılan bir handler'a yönlendirilmelidir. P bitini sıfır bırakmak, beklenmeyen bir vektörü teşhis edilebilir bir mesaj yerine #NP'ye ve büyük olasılıkla #DF'e dönüştürür. Hatanın kendisi yerine hatanın sonucunu görmek, aramayı yanlış yere yöneltir.

**Hizalama aritmetiğini bir kere yapın.** İşlemci çerçeveyi itmeden önce RSP'yi 16'nın katına hizalar. Hata kodu itmeyen bir vektörde beş qword itilir ve RSP 16'ya bölündüğünde 8 kalanını verir; hata kodu itilen vektörde altı qword ile kalan sıfır olur. Giriş kodu sahte hata kodunu ittiğinde her iki durumda da itilen qword sayısı yediye çıkar, on beş genel amaçlı register eklendiğinde toplam 22 qword (176 bayt) olur ve `call` anında RSP yeniden 16'nın katıdır. System V AMD64 ABI çağrı anında bu hizalamayı şart koşar; sağlanmadığında hata C kodunun kendisinde değil, derleyicinin ürettiği hizalı SSE erişimlerinde #GP olarak görünür.

**DPL'i varsayılan olarak sıfır yapın.** Kullanıcı modunun bilerek kullanmasını istediğiniz vektörler dışında (`int 0x80` biçiminde bir system call girişi, kesme noktası için vektör 3) tüm girdiler DPL = 0 olmalıdır. Vektör 14'ün DPL'i 3 bırakılırsa kullanıcı programı `int 14` çalıştırarak page fault handler'ına girebilir; işlemci bu yolda hata kodu itmez, CR2 eski değerini taşır ve handler stack'i yanlış okur. Bu, kullanıcının kernel'e istediği yanlış bilgiyi verebilmesi demektir.

**Kernel'i doğru bayraklarla derleyin.** `-mno-red-zone` IDT ile doğrudan ilgilidir: red zone RSP'nin altındaki 128 baytlık alandır ve işlemci interrupt çerçevesini tam oraya yazar. Derleyicinin SIMD register'larını kendiliğinden kullanmasını engellemek için `-mgeneral-regs-only` da gereklidir. Ayrıntılar [x86-64 register'ları](../03-assembly/registerlar.md) notundadır.

**`interrupt` niteleyicisi bir seçenektir, ama sınırlıdır.** GCC ve Clang, `__attribute__((interrupt))` ile işaretlenen fonksiyonlar için çerçeveyi kendisi ele alır ve `iretq` üretir; hata kodu ikinci parametre olarak alınabilir. Küçük kernel'lerde assembly yazmadan başlamak için makuldür. Ancak vektör numarasını bu yolla öğrenmenin yolu yoktur (her vektöre ayrı fonksiyon gerekir), `swapgs` yerleştirmesi üzerinde denetim vermez ve context switch için gereken tam register bağlamını ortaya çıkarmaz. Scheduler eklendiğinde assembly giriş koduna dönmek gerekir.

**Erken aşamada her exception'ı bastırmadan yazdırın.** Vektör numarası, hata kodu, RIP, CS, RFLAGS ve #PF'te CR2 — bu altı değer, kernel geliştirmenin ilk aylarındaki hataların büyük kısmını tek başına açıklar. Çıktı yolu heap'e ya da sürücü altyapısına bağımlı olmamalıdır; hata tam da o bileşenler bozulduğunda oluşur.

**Emülatörü teşhis aracı olarak kullanın.** QEMU'da `-d int` her teslim edilen vektörü, çerçeveyi ve hata kodunu yazar; `-no-reboot -no-shutdown` triple fault'ta makineyi yeniden başlatmak yerine durdurur ve `-d int,cpu_reset` ile birlikte zincirin nerede koptuğu görülebilir. IDT'si henüz çalışmayan bir kernel'de bu bayraklar, kernel'in kendi çıktısının yerini tutar.

## Mimariye göre farklar

| Mimari | Tabloyu gösteren | Tablonun içeriği | Dönüş | Olay bilgisi |
| --- | --- | --- | --- | --- |
| x86-64 | IDTR (`lidt`) | 256 adet 16 baytlık gate; her biri bir adres | `iretq` | vektör numarası, hata kodu, #PF'te CR2 |
| ARM64 | VBAR_EL1 | 16 giriş; her biri 128 baytlık **kod** alanı | `eret` | ESR_EL1, FAR_EL1 |
| RISC-V | stvec | tek giriş adresi ya da vektörlü modda taban | `sret` | scause, stval |

Fark isimlendirmenin ötesindedir. x86-64'te tablo adres tutar ve işlemci çerçeveyi stack'e iter. ARM64'te tablo doğrudan koddur: VBAR_EL1'in gösterdiği 2 KiB'lik alan dört gruba ayrılır (aynı EL ve SP_EL0, aynı EL ve SP_ELx, alt EL'den AArch64, alt EL'den AArch32) ve her grup senkron, IRQ, FIQ ve SError girişlerini içerir. Her girişe 128 bayt ayrılmıştır; oraya bir adres değil, çalıştırılacak komutlar yazılır. İşlemci stack'e bir şey itmez: dönüş adresi ELR_EL1'e, eski durum SPSR_EL1'e, olayın nedeni ESR_EL1'e konur. Kaydetme işinin tamamı yazılıma aittir.

RISC-V daha da azını yapar. stvec tek bir taban adresi ve iki bitlik bir mod alanı taşır; doğrudan modda tüm trap'ler aynı adrese gider, vektörlü modda yalnızca asenkron interrupt'lar taban + 4 × neden adresine dallanır. Neden scause'da, ilgili adres ya da komut stval'de, dönüş adresi sepc'te durur. Vektör başına ayrı bir handler adresi kavramı yoktur; ayrıştırma kernel'in ilk işidir.

Bu üçü arasında taşınabilir kod yazmanın yolu, giriş katmanını mimariye özgü bırakıp C tarafındaki dağıtım fonksiyonunu ortak tutmaktır.

## Sık yapılan hatalar

- Limiti tablonun boyutu olarak yazmak. IDTR'nin limit alanı boyuttan bir eksiktir; bir fazla yazmak son girdiden sonrasını da tabloya dahil eder.
- Hata kodu iten ve itmeyen vektörleri aynı giriş kodundan geçirmek. Sahte hata kodu itilmezse çerçeve sekiz bayt kayar; `iretq` yanlış adrese döner ve hata kesmenin oluştuğu yerde değil, rastgele bir yerde görünür.
- `iretq` yerine `ret` ile dönmek. `ret` yalnızca RIP'i çeker; CS, RFLAGS ve stack geri yüklenmez, ayrıcalık seviyesi kullanıcı moduna inmez.
- Hata kodunu ve vektör numarasını dönmeden önce stack'ten atmamak. `iretq` çerçevenin tam olarak beklediği yerde başlamasını ister.
- `swapgs`'i ayrıcalık seviyesi değişmediği halde çalıştırmak. Kernel modunda oluşan bir exception'da takas yapmak, işlemci başına veriye erişimi geçersiz bir adrese yönlendirir. Karar, itilen CS'in düşük iki bitine bakılarak verilir.
- PIC'i yeniden eşlemeden `sti` çalıştırmak. PIC'in varsayılan vektörleri 0–15 aralığındadır ve exception vektörleriyle çakışır; klavye kesmesi #GP olarak görünür.
- IDT'yi kurmadan önce uzun bir başlatma kodu yazmak. O aşamada oluşan her hata triple fault'a döner ve nedenini gösteren hiçbir iz kalmaz.
- Aynı IST girdisini birden çok vektöre vermek ya da IST stack'ini iç içe girilebilir sanmak. İkinci giriş birincinin çerçevesini siler.
- Handler adresinin sayfa tablosunda haritalı olduğunu varsaymak. Kernel'in kendi sayfa tablolarına geçildiğinde handler'ların bulunduğu bölge haritalanmamışsa ilk interrupt #PF ile başlayıp #DF ile biter.

## İlgili notlar

- [04 — Kernel](README.md) — bölümün diğer konuları
- [Kernel nedir](kernel-nedir.md) — kernel moduna giriş yollarına genel bakış
- [x86-64 register'ları](../03-assembly/registerlar.md) — RFLAGS, `swapgs` ve bağlam kaydetme
- [03 — Assembly](../03-assembly/) — interrupt giriş noktaları ve çağrı kuralı
- [05 — Memory](../05-memory/) — page fault'un ele alınması
- [07 — Drivers](../07-drivers/) — kesme tabanlı sürücü tasarımı

## Kaynaklar

- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 3A, Bölüm 6.10 — Interrupt Descriptor Table (IDTR, limit, tablonun konumu)
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 3A, Bölüm 6.11 — IDT Descriptors (gate biçimi, tipler, DPL)
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 3A, Bölüm 6.12 — Exception and Interrupt Handling (teslim sırası, DPL denetimi, RFLAGS'e etkisi)
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 3A, Bölüm 6.13 — Error Code
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 3A, Bölüm 6.14 — Exception and Interrupt Handling in 64-bit Mode (16 baytlık gate, çerçeve düzeni, IST)
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Cilt 2A ve 2B — INT n/INTO/INT3/INT1, IRET/IRETD/IRETQ, LIDT/SIDT komut tanımları
- AMD64 Architecture Programmer's Manual, Volume 2: System Programming, Bölüm 8 — Exceptions and Interrupts
- Arm® Architecture Reference Manual for A-profile architecture, Bölüm D1 — AArch64 System Level Programmers' Model (exception vektör tablosu, VBAR_EL1, ESR_EL1)
- The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Bölüm 4 — Supervisor-Level ISA (stvec, scause, stval, sepc)
- System V Application Binary Interface, AMD64 Architecture Processor Supplement, Bölüm 3.2.2 — The Stack Frame (16 baytlık hizalama, red zone)
- OSDev Wiki, "Interrupt Descriptor Table"; <https://wiki.osdev.org/Interrupt_Descriptor_Table> (erişim: 17 Eylül 2026)
