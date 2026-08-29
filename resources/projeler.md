# İncelenebilecek açık kaynak projeler

Bir mimari manual'ı mekanizmanın kurallarını verir, bir not o kuralların neden böyle olduğunu anlatır. İkisi de "bu mekanizma çalışan bir sistemde nereye oturuyor" sorusunu tam olarak yanıtlamaz. Sayfa tablosunun bayraklarını bilmek ile bir kernel'in fork sırasında o bayrakları hangi sırayla değiştirdiğini bilmek farklı şeylerdir. İkincisi yalnızca kaynak kodda görünür.

Bu liste, okunmak üzere seçilmiş açık kaynak sistemleri toplar. Ölçüt popülerlik değil okunabilirliktir: kodun izlenebilir olması, derlenip emülatörde çalıştırılabilmesi ve bir konuyu öğrenmek için makul bir giriş noktası sunması. Her girdide projenin ne olduğu, hangi konuları kapsadığı ve hangi seviyeye hitap ettiği belirtilir.

## Bir kernel kaynağı nasıl okunur

Kaynak kodu baştan sona okumak işe yaramaz. Kernel'ler dizin ağacında değil, çağrı yollarında okunur.

- **Tek bir yürütme yolu seçin ve sonuna kadar izleyin.** İyi başlangıçlar: bootloader'ın devrettiği giriş noktasından ilk sayfa tablosunun kurulmasına kadar olan yol, ya da bir `read()` çağrısının system call girişinden sürücüye ve geri dönüşe kadar geçtiği yol.
- **Önce derleme sistemine ve linker script'e bakın.** Kernel'in hangi adrese yerleştiğini, hangi bölümlerin nereye gittiğini ve hangi derleyici bayraklarının kullanıldığını bunlar söyler. Bu bilgi olmadan giriş fonksiyonunun neden o adreste çalıştığı anlaşılmaz.
- **Git geçmişini kaynak olarak kullanın.** Olgun bir projenin bugünkü hali, öğrenmek için çoğu zaman en kötü halidir. İlk commit'ler aynı mekanizmanın en sade sürümünü içerir. `git log --follow <dosya>` ve `git log -S<sembol>` bir özelliğin ne zaman ve neden eklendiğini gösterir.
- **Kodu okurken emülatörde çalıştırın.** QEMU'yu GDB ile durdurup register'lara ve sayfa tablosuna bakmak, aynı kodu yalnızca okumaktan daha hızlı öğretir.
- **Küçük bir sistemle başlayın.** Linux'un `mm/` dizinine ilk gün girmek öğrenmeyi hızlandırmaz.
- **Lisansı okuyun.** Kaynak koddan öğrenmek serbesttir; kod parçasını kendi projenize kopyalamak lisansa tabidir. Permissive lisanslarda (MIT, BSD) telif bildirimini korumak gerekir; GPL kodunu kopyalamak kendi projenizi de bağlar. Bir tasarımı anlayıp kendi kodunuzu yazmak ile kod taşımak farklı şeylerdir.

## Girdilerdeki alanlar

Her girdinin başında dört alan bulunur:

- **Dil** — kaynağın ağırlıklı olarak yazıldığı dil.
- **Lisans** — deponun kendi belirttiği lisans. Alt bileşenler farklı lisanslarda olabilir.
- **Mimari** — projenin desteklediğini belirttiği hedefler.
- **Seviye** — okumaya başlamak için gereken birikim. *Başlangıç*: C ve temel assembly bilgisi yeterlidir. *Orta*: paging, interrupt ve system call kavramlarının bilinmesi gerekir. *İleri*: sürücü, SMP ve dosya sistemi konularında ön bilgi olmadan yol izlemek zorlaşır.

## Baştan yazılmış tam sistemler

Bu projelerin ortak özelliği, kernel'in yanında dosya sistemi, sürücüler ve userspace tarafını da içermesidir. Bir bileşenin diğerine nasıl bağlandığını görmek için uygundurlar.

### Tunix

**Dil:** C · **Lisans:** MIT · **Mimari:** x86-64 · **Seviye:** İleri

**Depo:** <https://github.com/tunixos/tunix>

Sıfırdan yazılmış monolithic bir x86-64 kernel'i. Ayırt edici yanı, kernel'in üstündeki hiçbir şeyin bu projede yazılmamış olmasıdır: sistem, değiştirilmemiş bir Void Linux userspace'ini çalıştırır. İmaj Void'in kendi glibc paketlerinden, Void'in kendi paket yöneticisiyle (xbps) kurulur ve PID 1 olarak Void'in runit'i çalışır. Bu ayrım aynı zamanda bir test yöntemidir: bir bileşen çalışmadığında hata kernel'dedir, çünkü karşı taraf kernel'e göre uyarlanmamıştır.

Kernel yaklaşık 40 bin satır C ve az miktarda assembly içerir. Kapsamı: copy-on-write fork ile sanal bellek, firmware'in bildirdiği her işlemci üzerinde preemptive zamanlama, dinamik bağlayıcı destekli ELF yükleme, sinyaller ve Linux uyumlu bir system call tablosu. Sürücü tarafında IDE, AHCI, NVMe ve USB yığın depolama üzerine kurulu bir blok katmanı, GPT ve MBR bölümleri, `mkfs.ext2` ile üretilmiş okuma-yazma bir ext2 kök dosya sistemi, RTL8139 ile IPv4, ARP, ICMP, UDP ve TCP yığını, virtio-gpu, Intel HD Audio ve Ctrl+Alt+F1..F8 ile geçilen sekiz sanal terminal bulunur. Açılış Limine ile, BIOS veya UEFI üzerinden, tek bir GPT imajından yapılır.

Okumaya değer olduğu yer, Linux uyumluluğunun nerede bittiğidir. Void'in userspace'i `/proc`, `/sys`, `epoll`, seatd ve udevd gibi şeylerin belirli biçimlerde davranmasını bekler; kernel bu beklentileri karşılamak zorundadır. Bir Wayland compositor'ünün (Weston) bir sanal terminalden diğerine taşınabilmesi için gereken devir teslim protokolü, bu tür bir zorunluluğun somut örneğidir. `docs/` dizini bileşenleri ayrı ayrı anlatır; kaynağa girmeden önce oradan başlamak mantıklıdır. Kaynak ağacında `kernel/*.c` çekirdeği (system call'lar, process, bellek), `kernel/arch/x86_64/` ise yalnızca tek bir işlemcide anlam taşıyan kısmı barındırır.

### Managarm

**Dil:** C++, Rust, assembly · **Lisans:** MIT · **Mimari:** x86-64, aarch64, riscv64 · **Seviye:** İleri

**Depo:** <https://github.com/managarm/managarm>

Tamamen asenkron bir I/O modeli üzerine kurulmuş mikrokernel tabanlı bir sistem. Mikrokernel tasarımlarına yöneltilen klasik itiraz performanstır: her sürücü ayrı bir process olduğunda I/O yolu mesajlaşmayla dolar. Managarm bu maliyeti, kernel ve sunucuların arayüzünü baştan asenkron tanımlayarak azaltmayı hedefler; senkron bir çağrının bloke ettiği yerde asenkron model işlemleri sıraya alır.

Sistem mikrokernel olmasına rağmen Linux seviyesinde bir userspace uyumluluğu sunar: `epoll`, `signalfd`, `timerfd` ve `/proc`, `/sys` gibi sözde dosya sistemleri desteklenir, GNU/POSIX araç zinciri çalışır. Donanım tarafında 64 bit SMP ve ACPI, USB 3, NVMe, AHCI, Intel, NVIDIA ve virtio grafik sürücüleri, ağ ve seri aygıt desteği bulunur; KDE Plasma 6 ve Weston çalıştırılabilecek düzeydedir.

Bu listede iki soruya birlikte yanıt veren tek proje büyük olasılıkla budur: bir mikrokernel'de sürücüler ve dosya sistemleri userspace'e taşındığında system call yolu nasıl görünür, ve bu tasarımın üstünde Linux'a alışmış bir userspace nasıl ayakta tutulur. Mikrokernel ile monolithic kernel ayrımını yalnızca tanım düzeyinde bilen bir okuyucu için, farkın somut karşılığını görmenin en doğrudan yollarından biridir.

### SerenityOS

**Dil:** C++ · **Lisans:** BSD-2-Clause · **Mimari:** x86-64, ARM, RISC-V · **Seviye:** Orta

**Depo:** <https://github.com/SerenityOS/serenity>

Kernel'den masaüstü uygulamalarına kadar her şeyin sıfırdan yazıldığı Unix benzeri bir sistem. Dış bağımlılık kullanmama kararı, tek bir depo içinde kernel'in, standart kütüphane yerine geçen kendi kütüphanelerinin, pencere yöneticisinin ve uygulamaların birlikte bulunması anlamına gelir. Kod tabanı büyüktür ama tutarlıdır; `Kernel/`, `Userland/` ve `Libraries/` ayrımı ilk bakışta anlaşılır.

Uzun süre projenin parçası olan tarayıcı, Ladybird adıyla ayrı bir projeye taşınmıştır ve artık SerenityOS dışındaki sistemlerde de çalışır. Bu ayrılma, deponun bugünkü halini okurken akılda tutulmalıdır; tarayıcı motoruyla ilgili eski yazılar artık ayrı bir depoyu işaret eder.

Okumak için en uygun olduğu yer, kernel ile userspace arasındaki sınırdır. Bir system call'ın kernel tarafındaki karşılığı ile onu çağıran kütüphane kodu aynı depoda olduğu için iki ucu birlikte izlemek mümkündür.

### ToaruOS

**Dil:** C · **Lisans:** NCSA (BSD benzeri) · **Mimari:** x86-64, ARMv8 · **Seviye:** Orta

**Depo:** <https://github.com/klange/toaruos>

2011'den beri geliştirilen, çalışma zamanında hiçbir üçüncü taraf bağımlılığı olmayan bağımsız bir sistem. Kernel'i (Misaka) hybrid modüler bir tasarımdır ve 2020-2021'de SMP desteğiyle birlikte yeniden yazılmıştır. Sistemde bir compositor (Yutani), C standart kütüphanesi, dinamik bağlayıcı, terminal emülatörü, metin düzenleyici ve Kuroko adlı gömülü bir betik dili bulunur.

Bu listedeki daha büyük sistemlere göre okunması belirgin biçimde kolaydır. Özellikle iki konu için iyi bir kaynaktır: bir compositor'ün kernel'den ne beklediği (framebuffer, giriş aygıtları, paylaşımlı bellek) ve kendi libc'sini yazan bir sistemin dinamik bağlayıcıyı nasıl kurduğu.

### Redox

**Dil:** Rust · **Lisans:** MIT · **Mimari:** x86-64; aarch64 portu sürüyor · **Seviye:** Orta

**Depo:** <https://gitlab.redox-os.org/redox-os/redox> · **GitHub aynası:** <https://github.com/redox-os/redox>

Rust ile yazılmış, mikrokernel tabanlı bir sistem. Yalnızca bir kernel değildir; dosya sistemi, ekran sunucusu ve temel kullanıcı araçları da projenin parçasıdır. Tasarımında seL4, MINIX, Plan 9, Linux ve BSD'den etkilendiğini belirtir.

İki bakımdan öğreticidir. Birincisi, kernel yazarken belleğe dair güvenlik garantilerinin dil düzeyinde ne kadarının korunabildiği, ne kadarının `unsafe` bloklara düştüğüdür; sayfa tablosu ve fiziksel bellek yönetimi kodunda bu sınır açıkça görülür. İkincisi, Plan 9'dan gelen "her şey bir şema altında adreslenir" yaklaşımının kaynak yönetimine nasıl uygulandığıdır.

### skiftOS

**Dil:** C++ · **Lisans:** LGPL-3.0 veya sonrası · **Mimari:** x86-64 · **Seviye:** Orta

**Depo:** <https://github.com/skift-org/skift>

Modern C++ ile yazılmış, capability tabanlı bir mikrokernel (Hjert) üzerine kurulu sistem. Kernel'in yanında kendi çekirdek kütüphanesi (Karm), bir arayüz çatısı, masaüstü ortamı ve bir tarayıcı motoru bulunur. Proje kendi tanımıyla erken aşamadadır ve günlük kullanım için değildir.

Capability tabanlı bir kernel'de yetkinin nasıl temsil edildiğini küçük bir kod tabanı üzerinde görmek isteyenler için uygundur. seL4'ün aynı konudaki kodu doğruluk kanıtlarının gereklerine göre yazıldığı için daha yoğundur; skiftOS aynı fikri daha okunur bir ölçekte gösterir.

### HelenOS

**Dil:** C11, C++14 · **Lisans:** BSD (bazı üçüncü taraf bileşenler GPL) · **Mimari:** sekiz işlemci mimarisi · **Seviye:** İleri

**Depo:** <https://github.com/HelenOS/helenos>

Sıfırdan tasarlanmış, taşınabilir, mikrokernel tabanlı bir multiserver sistem. İşletim sistemi işlevleri userspace bileşenlerine bölünmüştür ve bu bileşenler mesajlaşmayla haberleşir; amaç modülerlik ve hata dayanıklılığıdır. ARM tabanlı gömülü aygıtlardan 32 ve 64 bit masaüstü PC'lere, Itanium ve SPARC sunuculara kadar sekiz mimaride çalışır.

Taşınabilirlik bu projede sonradan eklenmiş bir özellik değil, baştan verilmiş bir karardır. Mimariye özgü kodun nerede başlayıp nerede bittiğini ve sekiz farklı hedefi taşıyan bir soyutlamanın nasıl göründüğünü incelemek için iyi bir örnektir. Multiserver tasarımda bir sürücünün çökmesinin neden sistemi düşürmediği de aynı kod üzerinde izlenebilir.

### Haiku

**Dil:** C++ · **Lisans:** MIT (bazı bileşenler farklı) · **Mimari:** x86-64 ve 32 bit x86 · **Seviye:** İleri

**Depo:** <https://github.com/haiku/haiku> · **Site:** <https://www.haiku-os.org/about/>

BeOS'tan esinlenen ve onun kavramlarını sürdüren, kişisel bilgisayar hedefli bir sistem. Kernel, sürücüler, userspace hizmetleri, arayüz araç takımı, grafik yığını ve masaüstü uygulamaları tek bir ekip tarafından birlikte geliştirilir; sistemin tutarlılığı bu ortak geliştirmeden gelir.

Kernel'in tasarımı yanıt verme süresine göre kurgulanmıştır ve baştan çok iş parçacıklı düşünülmüştür. Bu listedeki çoğu projeden farklı olarak Haiku uzun süredir sürdürülen ve gerçek donanımda kullanılan bir sistemdir; ölçekli bir kod tabanının nasıl bakım gördüğünü de gösterir. Katkılar GitHub üzerinden değil, projenin kendi inceleme sistemi üzerinden yürütülür.

## Öğretmek için yazılmış kernel'ler

Bu projelerin amacı çalışan bir ürün olmak değil, bir konuyu göstermektir. Kapsamları dardır ve bu kasıtlıdır.

### xv6-riscv

**Dil:** ANSI C · **Lisans:** MIT · **Mimari:** riscv64 · **Seviye:** Başlangıç

**Depo:** <https://github.com/mit-pdos/xv6-riscv>

Dennis Ritchie ve Ken Thompson'ın Unix Version 6'sının, RISC-V çok işlemcili sistemler için yeniden yazılmış hali. MIT'nin 6.1810 işletim sistemleri dersinde kullanılır; geliştiricilerinin belirttiği öncelik yeni özellik eklemek değil, sadeleştirmek ve netleştirmektir.

Bu listede tamamı birkaç gün içinde okunabilecek tek sistem büyük olasılıkla budur. Sayfa tablosu kurulumu, trap girişi, process tablosu, scheduler, dosya sistemi ve kabuk birkaç bin satır içinde ve birbirine bağlı biçimde bulunur. RISC-V'nin ayrıcalık modeli x86-64'e göre çok daha az tarihsel yük taşıdığı için kavramlar mimarinin ayrıntılarına gömülmez. Dersin kitabı, kaynak kodun bölüm bölüm açıklaması olarak okunabilir.

### Writing an OS in Rust

**Dil:** Rust · **Lisans:** MIT veya Apache-2.0 · **Mimari:** x86-64 · **Seviye:** Başlangıç

**Depo:** <https://github.com/phil-opp/blog_os> · **Yazı dizisi:** <https://os.phil-opp.com/>

Rust ile x86-64 üzerinde adım adım kernel yazan bir yazı dizisi ve ona eşlik eden kod. Kapsam: freestanding bir ikili üretmek, minimal kernel, VGA metin modu, test altyapısı, CPU exception'ları, double fault, donanım interrupt'ları, paging, heap ve allocator tasarımları, `async`/`await` ile çok görevlilik.

Kod, klasör yerine dal (branch) yapısıyla düzenlenmiştir: her yazının sonundaki durum `post-XX` adlı bir dalda durur. Bu, bir özelliğin eklenmesinden önceki ve sonraki hali karşılaştırmak için kullanışlıdır. Rust bilmeyen bir okuyucu için bile IDT kurulumu ve double fault bölümleri, aynı konunun başka anlatımlarından daha nettir.

### MINIX 3

**Dil:** C · **Lisans:** BSD tarzı · **Mimari:** x86, ARM · **Seviye:** Orta

**Depo:** <https://github.com/Stichting-MINIX-Research-Foundation/minix> · **Site:** <https://www.minix3.org/>

Güvenilirlik hedefiyle tasarlanmış mikrokernel tabanlı bir sistem. Kernel modunda yalnızca küçük bir mikrokernel çalışır; işletim sisteminin geri kalanı, birbirinden yalıtılmış kullanıcı modu process'leri olarak yürür. Sürücülerin ayrı process'ler olması, çöken bir sürücünün sistemi düşürmeden yeniden başlatılabilmesini sağlar. NetBSD paketleriyle uyumludur.

MINIX'in eğitimdeki yeri, Tanenbaum'un işletim sistemleri kitabının referans implementasyonu olmasından gelir; kitapla birlikte okunmak üzere tasarlanmıştır. Yeni bir sistem yazarken örnek almak için değil, mikrokernel tasarımının gerekçelerini kaynağıyla birlikte görmek için okunur.

### Little Kernel (LK)

**Dil:** C, assembly · **Lisans:** MIT · **Mimari:** ARM32, ARM64, RISC-V, x86-32, x86-64, m68k, MIPS · **Seviye:** Orta

**Depo:** <https://github.com/littlekernel/lk>

Küçük sistemler için tasarlanmış, SMP farkındalığı olan bir kernel. Tam anlamıyla yeniden girişli, çok iş parçacıklı ve preemptive bir çekirdek sunar; modüler derleme sistemi sayesinde yalnızca gereken bileşenler dahil edilir. Birçok Android aygıtında bootloader olarak kullanılmıştır.

Gömülü ve gerçek zamanlı tarafa bakan bir kernel'in masaüstü hedefli olanlardan nerede ayrıldığını görmek için uygundur: sanal bellek isteğe bağlıdır, scheduler gecikme sınırlarına göre tasarlanmıştır ve kod tabanı tek bir hedefe göre kırpılabilir.

## Doğrulama ve güvenlik odaklı kernel'ler

### seL4

**Dil:** C, assembly · **Lisans:** GPL-2.0 (kernel), BSD-2-Clause (kütüphaneler) · **Mimari:** x86-64, ARM, RISC-V · **Seviye:** İleri

**Depo:** <https://github.com/seL4/seL4> · **Site:** <https://sel4.systems/>

Güvenlik ve biçimsel doğrulama üzerine kurulmuş bir mikrokernel. Ayırt edici yanı, implementasyonunun soyut spesifikasyonuna uygunluğunun matematiksel olarak kanıtlanmış olmasıdır. Kanıtların kapsamı sınırlıdır ve varsayımları vardır: hangi sürüm, hangi platform ve hangi yapılandırma için ne kanıtlandığı, sistem hakkında konuşmadan önce okunması gereken kısımdır.

Kaynak kod, doğrulanabilirlik uğruna sıra dışı kısıtlamalar altında yazılmıştır: kernel içinde dinamik bellek ayırma yoktur, bellek yönetimi capability'ler aracılığıyla userspace'e devredilmiştir. Bu, bir kernel'in ne kadar küçültülebileceğini ve bunun karşılığında userspace'in ne kadar yük aldığını gösteren uç bir örnektir. Referans manual'ı, kernel'in tüm arayüzünü capability'ler üzerinden tanımladığı için tek başına okunmaya değer.

### Zircon (Fuchsia)

**Dil:** C++ · **Lisans:** BSD-3-Clause · **Mimari:** x86-64, ARM64, RISC-V · **Seviye:** İleri

**Dokümantasyon:** <https://fuchsia.dev/fuchsia-src/concepts/kernel>

Fuchsia'nın temel platformu: bir kernel ve sistemin ayağa kalkması ile donanıma erişmesi için gereken userspace bileşenleri. Tasarımı üç kavram üzerine kuruludur. Kernel nesneleri sistemin yönettiği temel yapıları temsil eder; handle'lar bu nesnelere yapılan referanslardır; system call'lar handle'lar üzerinden işlem yapar. System call'ların büyük bölümü bloke etmez; bekleme, açıkça bekleme amaçlı çağrılarla yapılır.

Bu model, Unix'in dosya tanımlayıcısı merkezli tasarımına alışmış bir okuyucu için karşılaştırma noktası sunar: aynı problemler (nesne ömrü, yetki devri, olay bekleme) farklı bir soyutlamayla çözülmüştür. Zircon'un kökeni Little Kernel'e dayanır; iki kod tabanını yan yana okumak, küçük bir gömülü kernel'in nasıl büyüdüğünü gösterir.

## Üretimde kullanılan büyük kernel'ler

Bu kod tabanları öğretmek için yazılmamıştır. Buna rağmen belirli bir alt sistemi görmek için gereken tek yer çoğu zaman burasıdır; okurken kapsamı daraltmak zorunludur.

### Linux

**Dil:** C, assembly · **Lisans:** GPL-2.0 · **Mimari:** yirmiden fazla · **Seviye:** İleri

**Depo:** <https://github.com/torvalds/linux>

Kaynak ağacına bir konu belirlemeden girmek zaman kaybıdır. İşe yarayan yaklaşım tek bir alt sistemi hedef almaktır: `Documentation/` altındaki ilgili belgeyi okuyup ardından o alt sistemin dizinine inmek. Sürücü yazarken bir aygıtın Linux'taki sürücüsünü okumak çoğu zaman aygıtın veri sayfasından daha hızlı yol gösterir, çünkü donanımın belgede yazmayan davranışları kodun içindeki geçici çözümlerde görünür.

İkinci kaynak geliştirme listesidir. Bir değişikliğin neden yapıldığı commit mesajında ve o yamanın liste tartışmasında yazar; `git log` ile bulunan bir commit'in `Link:` satırı genellikle tartışmaya götürür.

Lisans konusunda dikkatli olun: kernel GPL-2.0 altındadır. System call arayüzü için kullanıcı programlarını kapsamayan bir istisna vardır, ancak kernel kodunu kendi projenize kopyalamak projenizi bağlar.

### FreeBSD

**Dil:** C · **Lisans:** BSD-2-Clause · **Mimari:** x86-64, ARM64, RISC-V ve diğerleri · **Seviye:** İleri

**Depo:** <https://github.com/freebsd/freebsd-src>

Kernel ve temel sistem araçlarının tek bir depoda birlikte geliştirildiği bir Unix türevi. Kernel kaynağı `sys/` altında mimari, sürücü ve alt sistem ayrımıyla düzenlenmiştir.

Sanal bellek yöneticisi ve VFS katmanı, tasarım kararlarının kitap düzeyinde belgelenmiş olması nedeniyle okunmaya değerdir. Permissive lisansı, kod parçalarını kendi projesine almak isteyen okuyucular için Linux'a göre daha esnek bir konumdadır; yine de telif bildirimi korunmalıdır.

### OpenBSD

**Dil:** C · **Lisans:** ISC ve BSD · **Mimari:** çok sayıda · **Seviye:** İleri

**Depo:** <https://github.com/openbsd/src>

Sadelik ve güvenliğe verdiği öncelikle bilinen bir sistem. Kod tabanı, karmaşıklığı azaltmak amacıyla düzenli olarak budanır; bu yüzden aynı işi yapan kod çoğu sistemde olduğundan kısadır. Bir mekanizmanın en yalın çalışan halini görmek istendiğinde iyi bir referanstır.

Güvenlik tarafında `pledge` ve `unveil` gibi, bir process'in yapabileceklerini kendi isteğiyle daraltan arayüzler kernel tarafındaki karşılıklarıyla birlikte okunabilir. Bu arayüzlerin implementasyonu, yetki kısıtlamanın capability tabanlı sistemlerden farklı bir yolla nasıl yapılabileceğini gösterir.

## Boot ve firmware

Kernel'in çalışmaya başladığı ana kadar olan kısım çoğu anlatımda "bootloader kernel'i yükler" cümlesiyle geçilir. Bu projeler o cümlenin içini doldurur.

### Limine

**Dil:** C, assembly · **Lisans:** BSD-2-Clause · **Mimari:** IA-32, x86-64, aarch64, riscv64, LoongArch64 · **Seviye:** Orta

**Depo:** <https://github.com/limine-bootloader/limine>

Modern, çok protokollü bir bootloader ve açılış yöneticisi; aynı zamanda Limine boot protokolünün referans implementasyonu. BIOS ve UEFI firmware'lerinin ikisini de destekler. Linux, Limine, Multiboot 1 ve Multiboot 2 protokollerini yükleyebilir, zincirleme açılış yapabilir; MBR, GPT ve bölümlenmemiş ortamları, FAT12/16/32 ve ISO9660 dosya sistemlerini okuyabilir.

Kendi kernel'ini yazan biri için iki açıdan önemlidir. Birincisi, protokolün kernel'e neyi hangi biçimde teslim ettiğini (bellek haritası, framebuffer, modül listesi, higher half yerleşimi) spesifikasyonundan okumak, kernel'in ilk fonksiyonunu yazmak için gereken bilginin büyük kısmıdır. İkincisi, aynı deponun içinde hem BIOS hem UEFI yolunun bulunması, iki firmware arayüzü arasındaki farkı tek bir kod tabanında görme imkânı verir.

### GNU GRUB

**Dil:** C, assembly · **Lisans:** GPL-3.0 veya sonrası · **Mimari:** x86, x86-64, ARM, ARM64, RISC-V ve diğerleri · **Seviye:** Orta

**Site:** <https://www.gnu.org/software/grub/>

Yaygın kullanılan açılış yükleyicisi ve Multiboot spesifikasyonunun referans noktası. Kendi kernel'ini yazan çoğu kişi ilk açılışını GRUB ve Multiboot 2 ile yapar; `grub-mkrescue` ile üretilen ISO ilk günlerde en kısa yoldur.

Kaynağı okumak için en uygun kısım dosya sistemi ve aygıt soyutlamalarıdır: firmware'in sunduğu sınırlı okuma imkânı üzerine kurulmuş, kernel'inkinden çok daha basit bir dosya sistemi katmanı içerir. GPL-3.0 lisansı, kod taşımayı düşünenler için dikkat gerektirir.

### EDK II (TianoCore)

**Dil:** C, Python · **Lisans:** BSD-2-Clause-Patent (bileşenlere göre değişir) · **Mimari:** x86, x86-64, ARM, ARM64, RISC-V, LoongArch · **Seviye:** İleri

**Depo:** <https://github.com/tianocore/edk2>

UEFI ve PI (Platform Initialization) spesifikasyonlarının açık kaynak implementasyonu ve firmware geliştirme ortamı. QEMU'nun UEFI ile açılması için kullanılan OVMF imajı da bu depodan üretilir.

UEFI uygulaması yazan veya UEFI'nin sunduğu servisleri kernel'inden çağıran biri için, spesifikasyondaki bir tanımın gerçekte nasıl karşılandığını görmenin yoludur. Boot Services ile Runtime Services ayrımının ve `ExitBootServices` çağrısından sonra hangi yapıların geçerli kaldığının kod düzeyindeki karşılığı burada bulunur. Kod tabanı büyüktür ve kendi adlandırma gelenekleri vardır; spesifikasyonu yanına açmadan okumak zordur.

### OpenSBI

**Dil:** C · **Lisans:** BSD-2-Clause · **Mimari:** RISC-V · **Seviye:** Orta

**Depo:** <https://github.com/riscv-software-src/opensbi>

RISC-V SBI (Supervisor Binary Interface) spesifikasyonunun, M-mode'da çalışan platforma özgü firmware'ler için yazılmış referans implementasyonu. M-mode'daki platform firmware'i ile S-mode veya HS-mode'da çalışan bootloader, hypervisor ve işletim sistemleri arasındaki katmanı oluşturur.

RISC-V hedefleyen bir kernel yazarken bu katmanın ne yaptığını bilmek zorunludur: timer kurulumu, işlemcilerin başlatılması ve konsol çıkışı gibi işlemler kernel'den doğrudan donanıma değil, SBI çağrılarıyla yapılır. x86-64'te firmware devrettikten sonra büyük ölçüde çekilirken, RISC-V'de sistem çalışırken de devrede kalan bir katman vardır; kaynak bu farkı somutlaştırır.

### U-Boot

**Dil:** C · **Lisans:** GPL-2.0 veya sonrası · **Mimari:** ARM, ARM64, RISC-V, x86, MIPS, PowerPC ve diğerleri · **Seviye:** Orta

**Depo:** <https://github.com/u-boot/u-boot>

Gömülü sistemlerde yaygın kullanılan bootloader. PC dünyasının aksine, gömülü kartlarda firmware'in sunduğu standart bir arayüz çoğu zaman yoktur; kartın belleği, saatleri ve çevre birimleri bootloader tarafından kurulur. U-Boot bu işi çok sayıda kart için yapar.

Device tree'nin nasıl yüklendiğini, işlendiğini ve kernel'e nasıl aktarıldığını izlemek için uygun bir kaynaktır. ARM veya RISC-V hedefleyen bir kernel donanımı device tree üzerinden tanıyacaksa bu zincirin başlangıcı buradadır.

## Tek bir alt sistemi okumak için

Bütün bir sistem yerine tek bir bileşene odaklanan, kendi kernel'ine taşınabilir ölçekte projeler.

### musl

**Dil:** C · **Lisans:** MIT · **Mimari:** çok sayıda · **Seviye:** Orta

**Kaynak ağacı:** <https://git.musl-libc.org/cgit/musl/>

Linux system call arayüzünü hedefleyen bir C standart kütüphanesi implementasyonu. Tasarım hedefleri statik ve dinamik bağlamanın verimliliği, küçük kod tabanı, doğru kullanıldığında öngörülebilir davranış ve standartlara uygunluktur; proje bu hedeflere anlaşılması ve bakımı kolay kodla ulaşılacağını savunur. ISO C99 ve POSIX 2008 temel standartlarını kapsar, ayrıca Linux, BSD ve glibc uyumluluğu için standart dışı arayüzler içerir.

Kendi libc'sini yazan biri için en uygun referanstır. glibc aynı işlevleri çok daha fazla katman, tarihsel uyumluluk kodu ve makro ile yapar; musl'da bir fonksiyonun implementasyonu genellikle tek bir kısa dosyadadır. Bellek ayırıcı, iş parçacığı desteği ve dinamik bağlayıcı ayrı ayrı okunabilecek bölümlerdir.

### lwIP

**Dil:** C · **Lisans:** BSD-3-Clause · **Mimari:** bağımsız · **Seviye:** Orta

**Depo:** <https://github.com/lwip-tcpip/lwip>

Küçük bellek ayak izi hedefiyle yazılmış bir TCP/IP yığını. Gömülü sistemlerde yaygın kullanılır ve işletim sisteminden bağımsız çalışacak biçimde tasarlanmıştır; altındaki katmanla bir uyarlama arayüzü üzerinden konuşur.

Kendi kernel'inde ağ desteği yazan biri için iki kullanımı vardır: ya doğrudan uyarlanır, ya da TCP durum makinesinin, yeniden gönderim timer'larının ve tampon yönetiminin okunabilir bir örneği olarak incelenir. Linux'un ağ yığını aynı konuları çok daha fazla optimizasyonla ele aldığı için ilk okuma için uygun değildir.

### smoltcp

**Dil:** Rust · **Lisans:** 0BSD · **Mimari:** bağımsız · **Seviye:** Orta

**Depo:** <https://github.com/smoltcp-rs/smoltcp>

Heap kullanmadan çalışabilen, bağımsız (`no_std`) bir TCP/IP yığını. Tüm tamponlar önceden ayrılır; yığın kendi başına iş parçacığı veya timer gerektirmez, zamanı ve paket giriş çıkışını çağıran taraf sağlar.

Bu tasarım, ağ yığını ile kernel arasındaki arayüzü olağandışı biçimde net gösterir: yığının neye ihtiyaç duyduğu (zaman, paket, tampon) ile neyi kendi başına yaptığı ayrılmıştır. Rust ile kernel yazanlar için doğrudan kullanılabilir; diğerleri için arayüz tasarımı açısından okunmaya değerdir.

### FatFs

**Dil:** C · **Lisans:** BSD tarzı (tek maddeli) · **Mimari:** bağımsız · **Seviye:** Başlangıç

**Site:** <https://elm-chan.org/fsw/ff/00index_e.html>

Küçük gömülü sistemler için yazılmış FAT ve exFAT implementasyonu. Blok aygıtına erişim, kullanıcının sağladığı birkaç fonksiyona indirgenmiştir; dosya sistemi kodu bu fonksiyonların üzerinde durur.

FAT, yapısı en basit yaygın dosya sistemidir ve bir kernel'in ilk dosya sistemi olarak sık seçilir. FatFs; boot sektörünün okunmasından FAT zincirinin izlenmesine ve dizin girdilerinin ayrıştırılmasına kadar tüm yolu, başka hiçbir bağımlılık olmadan gösterir. Bu listedeki en küçük ve en hızlı okunabilecek projedir.

## İlgili notlar

- [Kaynaklar](README.md) — bu bölümdeki diğer kaynak listeleri
- [Nereden başlanır: yol haritası](../00-giris/yol-haritasi.md) — hangi aşamada hangi projeyi okumanın anlamlı olduğu
- [Geliştirme ortamı kurulumu](../00-giris/gelistirme-ortami.md) — bu projeleri derleyip emülatörde çalıştırmak için gereken araçlar
- [02 — Boot](../02-boot/) — boot protokolleri ve firmware arayüzleri

## Kaynaklar

- Tunix, README ve `docs/` dizini; <https://github.com/tunixos/tunix> (erişim: 29 Ağustos 2026)
- Managarm, proje README'si; <https://github.com/managarm/managarm> (erişim: 29 Ağustos 2026)
- SerenityOS, proje README'si; <https://github.com/SerenityOS/serenity> (erişim: 29 Ağustos 2026)
- ToaruOS, proje README'si; <https://github.com/klange/toaruos> (erişim: 29 Ağustos 2026)
- Redox OS, proje ve kernel depolarının README'leri; <https://github.com/redox-os/redox> (erişim: 29 Ağustos 2026)
- skiftOS, proje README'si; <https://github.com/skift-org/skift> (erişim: 29 Ağustos 2026)
- HelenOS, proje README'si; <https://github.com/HelenOS/helenos> (erişim: 29 Ağustos 2026)
- Haiku, "About Haiku"; <https://www.haiku-os.org/about/> (erişim: 29 Ağustos 2026)
- xv6-riscv, proje README'si; <https://github.com/mit-pdos/xv6-riscv> (erişim: 29 Ağustos 2026)
- Philipp Oppermann, "Writing an OS in Rust"; <https://os.phil-opp.com/> (erişim: 29 Ağustos 2026)
- MINIX 3, proje sitesi ve kaynak deposu; <https://www.minix3.org/> (erişim: 29 Ağustos 2026)
- Little Kernel, proje README'si; <https://github.com/littlekernel/lk> (erişim: 29 Ağustos 2026)
- seL4, proje README'si ve seL4 Reference Manual; <https://sel4.systems/> (erişim: 29 Ağustos 2026)
- Fuchsia dokümantasyonu, "Zircon kernel" — Kernel Objects, Handles ve System Calls; <https://fuchsia.dev/fuchsia-src/concepts/kernel> (erişim: 29 Ağustos 2026)
- Limine, proje README'si ve Limine Boot Protocol spesifikasyonu; <https://github.com/limine-bootloader/limine> (erişim: 29 Ağustos 2026)
- GNU GRUB Manual; <https://www.gnu.org/software/grub/> (erişim: 29 Ağustos 2026)
- TianoCore EDK II, proje README'si ve UEFI Specification; <https://github.com/tianocore/edk2> (erişim: 29 Ağustos 2026)
- OpenSBI, proje README'si ve RISC-V SBI Specification; <https://github.com/riscv-software-src/opensbi> (erişim: 29 Ağustos 2026)
- U-Boot, proje deposu ve dokümantasyonu; <https://github.com/u-boot/u-boot> (erişim: 29 Ağustos 2026)
- musl libc, kaynak ağacındaki README; <https://git.musl-libc.org/cgit/musl/> (erişim: 29 Ağustos 2026)
- lwIP, proje deposu; <https://github.com/lwip-tcpip/lwip> (erişim: 29 Ağustos 2026)
- smoltcp, proje README'si; <https://github.com/smoltcp-rs/smoltcp> (erişim: 29 Ağustos 2026)
- ChaN, "FatFs — Generic FAT Filesystem Module"; <https://elm-chan.org/fsw/ff/00index_e.html> (erişim: 29 Ağustos 2026)
