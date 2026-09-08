## EXT2 dosya sistemi nedir?

EXT2 (Second Extended Filesystem), Linux için geliştirilen ve 1993 yılında yayımlanan bir dosya sistemidir. İlk Extended Filesystem'ın (ext) ardılı olarak geliştirilmiş, daha sonra yerini büyük ölçüde ext3 ve ext4'e bırakmıştır.

EXT2'nin tasarımında performans, genişletilebilirlik ve Unix dosya sistemi özellikleri önemli rol oynar. Dosya izinleri, sahiplik bilgileri, sembolik bağlantılar ve inode tabanlı dosya yönetimi gibi geleneksel Unix özelliklerini destekler.

EXT2'nin önemli özelliklerinden biri **journaling kullanmamasıdır**. Bu nedenle dosya sistemi üzerinde yapılan değişiklikler ayrıca bir günlükte tutulmaz. Journaling'in olmaması bazı durumlarda daha düşük yazma yükü sağlasa da beklenmeyen güç kesintileri veya sistem çökmelerinden sonra dosya sistemi kontrolü gerekebilir.

EXT2'nin ardından geliştirilen ext3, temel olarak ext2 yapısına journaling desteği eklemiştir. ext4 ise daha büyük dosya sistemleri, extents, daha yüksek performans ve çeşitli yeni özellikler getirmiştir.

### EXT2 sınırları

EXT2'nin desteklediği maksimum dosya ve dosya sistemi boyutu kullanılan blok boyutuna ve işletim sistemi implementasyonuna bağlıdır. Bu nedenle EXT2 için tek bir sabit maksimum disk veya dosya boyutundan söz etmek doğru değildir.

Dosya ve dizin adları en fazla **255 bayt** uzunluğunda olabilir.

EXT2'nin eski dizin yapısında inode bağlantı sayısından kaynaklanan bazı alt dizin sınırları bulunur. Ancak bu sınır, “32.768 seviye iç içe dizin oluşturulabilir” şeklinde yorumlanmamalıdır.

EXT2 günümüzde masaüstü ve sunucu sistemlerinde genellikle ext4 gibi daha modern dosya sistemlerinin gerisinde kalmıştır. Bununla birlikte journaling gerekmeyen küçük sistemlerde, bazı gömülü sistemlerde ve eğitim amaçlı işletim sistemi projelerinde basit yapısı nedeniyle hâlâ kullanışlıdır.
