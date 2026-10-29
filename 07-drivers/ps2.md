# PS/2 klavye ve fare

PS/2, 1987'de IBM'in kişisel bilgisayarlarında kullanılmaya başlanan, klavye ve fareyi aynı denetleyici üzerinden bağlayan seri bir arayüzdür. Günümüzde yerini büyük ölçüde USB almış olsa da PC platformunda, özellikle QEMU ve eski donanımda, hâlâ en kolay erişilebilen giriş aygıtlarından biridir. Klavye ve fare aynı iki portu paylaştığı için sürücü, iki aygıtın verisini birbirine karıştırmadan ayırmak zorundadır.

## Ön koşullar

- [Donanım erişimi: port I/O ve MMIO](donanim-erisimi.md)
- [Kesme tabanlı sürücüler ve interrupt handler tasarımı](kesme-tabanli-suruculer.md)
- [PIC (8259)](../04-kernel/pic.md)
- [IDT (Interrupt Descriptor Table)](../04-kernel/idt.md)

## Kapsam

Bu not x86 PC'deki 8042 uyumlu klavye denetleyicisini, PS/2 klavyeyi ve üç baytlık standart PS/2 fareyi anlatır. Kod örnekleri C ile yazılmıştır ve `inb`/`outb` fonksiyonlarının tanımlı olduğu varsayılır. USB klavye ve fare bu notun kapsamı dışındadır; BIOS'un USB legacy emülasyonu sunması durumunda aygıt PS/2 gibi görünebilir.

## Problem

Klavye ve fare, kullanıcının ne zaman bir tuşa basacağını veya fareyi hareket ettireceğini önceden bilmediğimiz aygıtlardır. Sürekli port okumak CPU'yu boşa harcar. Bu yüzden ikisi de interrupt üretir: klavye IRQ 1, fare IRQ 12.

Ancak iki aygıt tek bir denetleyiciyi ve tek bir veri portunu paylaşır. Denetleyici, hangi baytın klavyeden hangisinin fareden geldiğini sürücüye söylemek için durum register'ında bir bit kullanır. Sürücü bu biti okumadan veriyi tüketirse klavye tarama kodunu fare paketi sanabilir veya tersini yapabilir. Üstelik fare paketi üç bayttır ve baytlar arasında interrupt gelebilir; paketin ortasında başka bir olay girerse senkronizasyon kaybolur.

Klavye tarafında da benzer bir sorun vardır. Tuşa basmak ve bırakmak ayrı tarama kodları üretir; basılı tutma ise typematic denen tekrar mesajları üretir. Sürücü, ham tarama kodlarını tuş olaylarına çevirmek, basılı tuşları takip etmek ve ölü tuşları yönetmek zorundadır. Bütün bunlar interrupt bağlamında yapılmamalıdır.

## Donanım seviyesinde

### 8042 denetleyicisi

Klavye ve fare, anakart üzerindeki klavye denetleyicisine bağlıdır. Tarihsel olarak Intel 8042 mikrodenetleyicisi kullanıldığı için bu adla anılır; modern yonga setlerinde aynı arayüzü taklit eden bir birim bulunur. İki port kullanılır:

| Port | Ad | Kullanım |
| --- | --- | --- |
| `0x60` | Veri | Klavye/fare verisi ve denetleyiciye gönderilen veri |
| `0x64` | Durum / komut | Okunduğunda durum, yazıldığında denetleyici komutu |

Durum register'ının önemli bitleri:

| Bit | Ad | Anlamı |
| --- | --- | --- |
| 0 | OBF (Output Buffer Full) | `0x60`'ta okunacak veri var |
| 1 | IBF (Input Buffer Full) | Denetleyici henüz önceki yazmayı almadı |
| 2 | Sistem bayrağı | Açılışta self-test geçildiyse 1 |
| 3 | Veri/komut | Son yazılan bayt veri mi (0), komut mu (1) |
| 4 | Klavye kilidi | Klavye kilidi durumu |
| 5 | Aux veri | `0x60`'taki veri fareden geldi |
| 6 | Zaman aşımı | Önceki işlem zaman aşımına uğradı |
| 7 | Parite hatası | Gelen baytta parite hatası var |

`0x60`'tan okumadan önce OBF biti beklenir. `0x60` veya `0x64`'e yazmadan önce IBF bitinin temiz olması beklenir. Bu beklemelerin her birine zaman aşımı konmalıdır; denetleyici yoksa veya arızalıysa döngü sonsuza kadar dönmemelidir.

### Denetleyici komutları

`0x64` portuna yazılan komutlar denetleyicinin kendisini yönetir:

| Komut | Anlamı |
| --- | --- |
| `0x20` | Yapılandırma baytını oku |
| `0x60` | Yapılandırma baytını yaz |
| `0xA7` | İkinci (fare) portunu kapat |
| `0xA8` | İkinci (fare) portunu aç |
| `0xAD` | Klavyeyi kapat |
| `0xAE` | Klavyeyi aç |
| `0xD4` | Bir sonraki `0x60` yazmasını fareye gönder |
| `0xAA` | Denetleyici self-test'i; başarılıysa `0x55` döner |
| `0xAB` | Arayüz testi; başarılıysa `0x00` döner |

Yapılandırma baytı, interrupt'ların ve saat hatlarının durumunu belirler. Önemli bitler:

| Bit | Anlamı |
| --- | --- |
| 0 | Klavye interrupt'ı açık |
| 1 | Fare interrupt'ı açık |
| 2 | Sistem bayrağı |
| 3 | Klavye kilidini yoksay |
| 4 | Klavye saatini kapat |
| 5 | Fare saatini kapat |
| 6 | Tarama kodu çevirisi (set 2 → set 1) |

Bit 6 set ise denetleyici, klavyenin gönderdiği set 2 tarama kodlarını set 1'e çevirir. Sürücü hangi seti beklediğini bu bitten okumalıdır. BIOS genellikle çeviriyi açık bırakır, ancak bu garanti değildir.

### Klavye komutları

Klavyeye komut göndermek için `0x60`'a bir bayt yazılır. Klavye her komuttan sonra `0xFA` (ACK) döner. Hata durumunda `0xFE` (resend) veya `0xFC` (error) gelebilir. Sık kullanılan komutlar:

| Komut | Anlamı |
| --- | --- |
| `0xFF` | Klavyeyi sıfırla; self-test sonucu `0xAA` döner |
| `0xF4` | Taramayı aç |
| `0xF5` | Taramayı kapat |
| `0xF3` | Typematic hızını ayarla; ardından bir bayt gelir |
| `0xED` | LED'leri ayarla; ardından LED baytı gelir |
| `0xEE` | Echo |
| `0xF0` | Tarama kodu setini seç; ardından set numarası gelir |
| `0xF2` | Klavye kimliğini oku |

`0xF3` sonrası gönderilen baytta düşük 5 bit tekrar hızını, yüksek 3 bit ise gecikmeyi belirler. `0xED` sonrası gönderilen baytta bit 0 scroll lock, bit 1 num lock, bit 2 caps lock LED'lerini kontrol eder.

### Fare komutları

Fareye komut göndermek için önce `0x64`'e `0xD4`, sonra `0x60`'a komut yazılır. Fare de her komuttan sonra `0xFA` ile onay verir. Sık kullanılan komutlar:

| Komut | Anlamı |
| --- | --- |
| `0xFF` | Fareyi sıfırla; self-test `0xAA`, ardından kimlik baytı |
| `0xF6` | Varsayılan ayarlara dön |
| `0xF4` | Veri raporlamayı aç |
| `0xF5` | Veri raporlamayı kapat |
| `0xF3` | Örnekleme hızını ayarla; ardından bir bayt |
| `0xE8` | Çözünürlüğü ayarla; ardından bir bayt |
| `0xE6` | 1:1 ölçekleme |
| `0xE7` | 2:1 ölçekleme |
| `0xF2` | Aygıt kimliğini oku |

Standart fare üç baytlık paket gönderir. Tekerlekli farelerde (Intellimouse) dördüncü bayt gelir; bunun için kimlik sorgusu ve örnekleme hızı dizisiyle aygıtın bu modu desteklediği anlaşılır. Bu not üç baytlık temel paketi ele alır.

### Tarama kodları

Klavye, her tuş için bir tarama kodu (scan code) gönderir. Set 1'de:

- Tuşa basıldığında "make" kodu gelir.
- Tuş bırakıldığında aynı kodun 7. biti set edilmiş "break" kodu gelir. Örneğin `0x1C` (Enter) bırakıldığında `0x9C` gelir.
- Genişletilmiş tuşlar `0xE0` önekiyle gelir. Örneğin sağ Ctrl `0xE0 0x1D`, bırakılışı `0xE0 0x9D` olur.
- `Pause` tuşu `0xE1` ile başlayan özel bir dizi kullanır.

Set 2'de make ve break kodları farklıdır; break çoğu tuşta `0xF0` önekiyle gelir. Denetleyicinin çevirisi açıksa sürücü set 1 görür. Çeviri kapalıysa set 2 tablosu kullanılmalıdır.

Klavye ayrıca basılı tutulan tuşlar için typematic mesajları üretir. Sürücü, basılı tuşları bir durum tablosunda tutmalı ve tekrar mesajlarını ayrı bir olay olarak ele almalıdır. Aksi halde her tekrar, yeni bir tuşa basılmış gibi görünür.

### Fare paketi

Üç baytlık fare paketinin ilk baytı bayrakları taşır:

| Bit | Anlamı |
| --- | --- |
| 0 | Sol tuş basılı |
| 1 | Sağ tuş basılı |
| 2 | Orta tuş basılı |
| 3 | Paket başlangıcı (her zaman 1) |
| 4 | X yönü negatif |
| 5 | Y yönü negatif |
| 6 | X taşması |
| 7 | Y taşması |

İkinci bayt X hareketi, üçüncü bayt Y hareketidir. Hareket değerleri 8 bitlik işaretli sayılardır; bayraklardaki işaret bitleriyle birlikte yorumlanır. PS/2'de pozitif Y yukarı yöndür; ekran koordinatlarında Y aşağı doğru arttığı için işaret çevrilir.

Paket başlangıç biti (bit 3) set değilse sürücü senkronizasyonu kaybetmiştir. Bu durumda paket atılmalı ve bir sonraki bayttan itibaren yeniden senkronize olunmalıdır. Taşma bitleri set ise o eksendeki hareket güvenilir değildir; paket yine de işlenebilir ama taşma raporlanmalıdır.

## Kernel tarafında

### Başlatma sırası

Tipik bir başlatma sırası şöyledir:

1. Klavye ve fare portlarını kapat.
2. Veri tamponunu boşalt.
3. Denetleyici yapılandırma baytını oku.
4. Gerekirse self-test ve arayüz testi yap.
5. Klavye ve fare portlarını aç.
6. Yapılandırma baytında interrupt'ları aç, saat hatlarını aç.
7. Klavyeyi `0xF4` ile taramaya başlat.
8. Fareyi `0xF6`, `0xF4` ile varsayılana dönüp raporlamayı aç.
9. IRQ 1 ve IRQ 12'yi interrupt denetleyicisinde aç.
10. Handler'ları kaydet.

Self-test (`0xAA`) bazı denetleyicilerde yapılandırma baytını sıfırlar. Bu yüzden yapılandırma baytı self-test'ten sonra yeniden okunmalı ve yazılmalıdır.

### Temel erişim yardımcıları

```c
#define PS2_VERI   0x60
#define PS2_DURUM  0x64
#define PS2_KOMUT  0x64

static int ps2_yazma_bekle(void)
{
    for (int i = 0; i < 100000; i++) {
        if (!(inb(PS2_DURUM) & 0x02))
            return 0;
    }
    return -1;   /* zaman aşımı */
}

static int ps2_okuma_bekle(void)
{
    for (int i = 0; i < 100000; i++) {
        if (inb(PS2_DURUM) & 0x01)
            return 0;
    }
    return -1;
}

static int ps2_komut_gonder(uint8_t komut)
{
    if (ps2_yazma_bekle() < 0)
        return -1;
    outb(PS2_KOMUT, komut);
    return 0;
}

static int ps2_veri_gonder(uint8_t veri)
{
    if (ps2_yazma_bekle() < 0)
        return -1;
    outb(PS2_VERI, veri);
    return 0;
}

static int ps2_veri_al(uint8_t *veri)
{
    if (ps2_okuma_bekle() < 0)
        return -1;
    *veri = inb(PS2_VERI);
    return 0;
}
```

Bu örnekler ACK ve resend akışını göstermez. Gerçek sürücü, her komuttan sonra gelen yanıtı okumalı ve `0xFE` gelirse komutu sınırlı sayıda yeniden denemelidir.

### Klavye interrupt handler'ı

```c
static struct halka klavye_kuyrugu;

static int klavye_irq(unsigned int irq, void *baglam)
{
    uint8_t durum = inb(PS2_DURUM);

    if (!(durum & 0x01))
        return IRQ_BENIM_DEGIL;

    uint8_t bayt = inb(PS2_VERI);

    /* Fare verisi klavye kuyruğuna girmemeli. */
    if (durum & 0x20)
        return IRQ_BENIM_DEGIL;

    if (!halka_yaz(&klavye_kuyrugu, bayt)) {
        klavye_kesmeyi_kapat();
        klavye_kuyrugu.tasti = 1;
    }

    return IRQ_ISLENDI;
}
```

Handler yalnızca ham baytı kuyruğa koyar. Tarama kodunu tuşa çevirmek, basılı tuş durumunu güncellemek ve terminal tamponuna yazmak alt yarıya bırakılır. Böylece handler kısa kalır ve klavye interrupt'ı sık gelse bile sistem boğulmaz.

### Fare interrupt handler'ı

Fare paketi üç bayt olduğu için handler bir indeks tutar. İlk baytın bit 3'ü set değilse paket baştan başlatılır:

```c
static uint8_t fare_paketi[3];
static int fare_indeks;

static int fare_irq(unsigned int irq, void *baglam)
{
    uint8_t durum = inb(PS2_DURUM);

    if (!(durum & 0x01) || !(durum & 0x20))
        return IRQ_BENIM_DEGIL;

    uint8_t bayt = inb(PS2_VERI);

    if (fare_indeks == 0 && !(bayt & 0x08)) {
        /* Senkronizasyon yok: bu baytı atla. */
        return IRQ_ISLENDI;
    }

    fare_paketi[fare_indeks++] = bayt;

    if (fare_indeks == 3) {
        fare_indeks = 0;
        fare_paketini_isle(fare_paketi);
    }

    return IRQ_ISLENDI;
}
```

`fare_paketini_isle` içinde işaret bitleri ve taşma bayrakları yorumlanır:

```c
static void fare_paketini_isle(const uint8_t *p)
{
    int16_t dx = p[1];
    int16_t dy = p[2];

    if (p[0] & 0x10)
        dx -= 256;   /* X negatif */
    if (p[0] & 0x20)
        dy -= 256;   /* Y negatif */

    /* PS/2'de Y yukarı pozitiftir; ekran koordinatında işareti çevir. */
    fare_olayi_uret(dx, -dy, p[0] & 0x07);
}
```

### Tarama kodundan tuşa geçiş

Ham tarama kodunu doğrudan ASCII'ye çevirmek eksik bir yaklaşımdır. Doğru yol, önce tuş kimliğine (keycode) geçmek, sonra klavye düzenine göre karakter üretmektir. Shift, Ctrl, Alt ve Caps Lock durumu bu dönüşümde kullanılır. Ölü tuşlar (dead key) ayrı bir durum makinesi gerektirir.

Türkçe klavye düzeni için harf eşlemesi Q veya F düzenine göre değişir. Sürücü katmanı mümkünse yalnızca tuş olayı üretmeli; düzen eşlemesi kullanıcı alanındaki klavye yapılandırmasına bırakılmalıdır. Aksi halde aynı kernel farklı ülkelerde yanlış karakter üretir.

## Implementasyon notları

**İki aygıtın verisini durum bitiyle ayırın.** `0x60`'tan okunan baytın klavyeye mi fareye mi ait olduğu, okuma anındaki durum register'ının 5. bitinden anlaşılır. Önce veriyi okuyup sonra durumu okumak yarışa yol açabilir; durumu ve veriyi mümkün olduğunca yakın okuyun.

**Her erişime zaman aşımı koyun.** 8042 emülasyonu bozuk veya aygıt yoksa OBF/IBF bitleri hiç değişmeyebilir. Zaman aşımında sürücü aygıtı devre dışı bırakmalı ve hatayı bildirmelidir.

**ACK ve resend'i yönetin.** Her klavye/fare komutu `0xFA` bekler. `0xFE` gelirse komut yeniden gönderilmelidir; sonsuz döngüye karşı deneme sayısı sınırlanmalıdır. `0xFC` kalıcı hata olarak ele alınmalıdır.

**Başlatmada tamponu boşaltın.** Açılışta denetleyicide eski veriler kalmış olabilir. Portları kapatıp tamponu boşaltmadan yapılandırma yazmak, eski bir baytı komut yanıtı sanmanıza yol açar.

**Klavye ve fare komutlarını sıraya koyun.** İki aygıt aynı denetleyiciyi paylaşır. Aynı anda klavye komutu ve fare verisi göndermek, yanıtların karışmasına neden olur. Sürücü seviyesinde bir kilit veya komut kuyruğu gerekir.

**Paket ortasında gelen baytları doğru ele alın.** Fare paketi üç bayttır; interrupt her baytta gelebilir. İndeks ve senkronizasyon biti olmadan paketleri birleştirmek, imlecin zıplamasına yol açar.

**Taşma ve hata bayraklarını yoksaymayın.** Fare taşması, hızlı hareket sırasında delta değerinin 8 bitte ifade edilemediğini gösterir. Sürücü bunu raporlamalı veya hareketi sınırlamalıdır.

**Klavye düzenini kernel'e gömmeyin.** Tuş olayı ile karakter dönüşümünü ayırın. Bu, birden çok dil düzenini ve kullanıcı tercihini desteklemenin tek makul yoludur.

**USB legacy emülasyonunu unutmayın.** BIOS, USB klavyeyi PS/2 gibi gösterebilir. Bu durumda aygıt gerçek bir 8042 değildir ve bazı komutlar farklı davranabilir. Sürücü, desteklenmeyen komutlarda takılmamalıdır.

**ACPI ile denetleyicinin varlığını doğrulayın.** FADT'teki Boot Architecture Flags alanındaki 8042 biti, platformda klavye denetleyicisi olup olmadığını gösterir. Ayrıntı `acpi.md` notunun konusudur.

## Mimariye göre farklar

| Mimari | PS/2 desteği |
| --- | --- |
| x86-64 | Standart 8042 uyumlu denetleyici; QEMU `pc` ve `q35` makinelerinde bulunur |
| ARM64 | Genellikle yok; giriş USB HID veya platforma özgü aygıtlarla yapılır |
| RISC-V | Genellikle yok; giriş USB HID veya platforma özgü aygıtlarla yapılır |

ARM64 ve RISC-V'de PS/2 denetleyicisi standart değildir. QEMU `virt` gibi platformlarda giriş genellikle USB HID veya seri konsol üzerinden yapılır. Bu yüzden taşınabilir bir giriş katmanı, PS/2'yi yalnızca bir arka uç olarak ele almalı; üst katmana ham tarama kodu yerine tuş olayı sunmalıdır.

## Sık yapılan hatalar

- Durum register'ının 5. bitine bakmadan fare ve klavye verisini karıştırmak.
- OBF/IBF beklemesine zaman aşımı koymamak.
- Komut sonrası ACK baytını okumamak; sonraki veriyi ACK sanmak.
- Fare paketini bayt sınırlarına göre değil, interrupt sınırlarına göre birleştirmek.
- Fare paketinin 3. bitini kontrol etmeden senkronizasyonu kaybetmek.
- PS/2 Y ekseninin yönünü ters çevirmemek.
- Klavye tarama kodunu doğrudan ASCII sanmak.
- Klavye ve fare komutlarını kilitsiz aynı anda göndermek.
- `0xAA` self-test sonrası yapılandırma baytını yeniden yazmamak.
- USB legacy emülasyonunu gerçek 8042 sanmak.

## İlgili notlar

- [Kesme tabanlı sürücüler ve interrupt handler tasarımı](kesme-tabanli-suruculer.md)
- [Donanım erişimi: port I/O ve MMIO](donanim-erisimi.md)
- [Sürücü nedir, kernel içinde nerede durur](surucu-nedir.md)
- [PIC (8259)](../04-kernel/pic.md)
- [IDT (Interrupt Descriptor Table)](../04-kernel/idt.md)
- [Seri port (UART)](uart.md)

## Kaynaklar

- IBM, *Personal System/2 Hardware Interface Technical Reference* — Keyboard and Auxiliary Device Controller
- Intel 8042 Programmable Peripheral Interface veri sayfası
- OSDev Wiki, "PS/2 Keyboard"; <https://wiki.osdev.org/PS/2_Keyboard> (erişim: 29 Ekim 2026)
- OSDev Wiki, "PS/2 Mouse"; <https://wiki.osdev.org/PS/2_Mouse> (erişim: 29 Ekim 2026)
- OSDev Wiki, "8042"; <https://wiki.osdev.org/8042> (erişim: 29 Ekim 2026)
- ACPI Specification, FADT, Boot Architecture Flags; <https://uefi.org/specifications> (erişim: 29 Ekim 2026)
