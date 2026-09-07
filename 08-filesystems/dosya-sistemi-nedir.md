# Dosya Sistemi (File System) Nedir?

**Dosya sistemi;** bir depolama cihazındaki (SSD, HDD, USB bellek vb.) verilerin nasıl adlandırılacağını, nerede saklanacağını, nasıl bulunacağını ve nasıl organize edileceğini belirleyen kurallar ve veri yapıları bütünüdür.

## Bir Dosya Sistemi Olmasaydı Ne Olurdu?
Depolama biriminiz 0 ve 1'lerden oluşan, devasa bir dağılmış oda gibi olurdu. İşletim sisteminiz bir dosyanın nerede başlayıp nerede bittiğini veya hangi klasöre ait olduğunu anlayamazdı.

## Temel Görevleri
- **Organizasyon:** Sisteminizdeki dosyaları ve klasörleri hiyerarşik (ağaç yapısında) bir düzene sokar.
- **Erişim ve Güvenlik Yönetimi:** Dosyalara okuma, yazma ve çalıştırma izinleri atayarak yetkisiz erişimleri engeller.
- **Alan Yönetimi:** Diskteki boş ve dolu alanları (sektör/blok bazında) takip eder.
- **Adresleme:** Her dosyaya sistem içinde benzersiz bir dosya yolu (path) atar.
- **Veri Bütünlüğü:** Günlükleme (journaling) özelliği sayesinde ani güç kesintilerinde veri bozulmalarını önler.

## En Yaygın Dosya Sistemleri
- **NTFS:** Windows'un varsayılan dosya sistemidir. Büyük dosya desteği, gelişmiş güvenlik izinleri ve günlükleme sağlar.
- **FAT32:** USB bellekler ve eski cihazlarda kullanılır. Neredeyse tüm işletim sistemleriyle uyumludur ancak tek parça halinde maksimum 4 GB dosya boyutunu destekler.
- **exFAT:** FAT32'nin gelişmiş halidir. 4 GB dosya boyutu limitini kaldırır. USB bellekler ve SD kartlarda yaygın olarak kullanılır; hem Windows hem macOS üzerinde sorunsuz çalışır.
- **ext4:** Linux ve Android sistemlerde kullanılır. Yüksek performansa sahip olup gelişmiş günlükleme (journaling) mekanizmasını destekler.
- **APFS:** Apple cihazlarına (macOS, iOS) özeldir. SSD'ler için optimize edilmiştir; çok hızlı dosya kopyalama ve güçlü şifreleme özellikleri sunar.
