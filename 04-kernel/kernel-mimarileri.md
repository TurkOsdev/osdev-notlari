# Kernel (çekirdek) Mimarisi nedir?

* **Kernel (çekirdek) mimarisi, bir uygulama veya programın bilgisayar donanımı ile köprüyü oluşturan ana bileşenin sistem kaynaklarını yönetmek ve optimize etmek için tasarlanan yol'a (path) denir.**

## Temel Görevleri
* **Donanım yönetimi:** CPU, RAM, SSD, Klavye & Mouse vb. donanımların bir arada stabil çalışmasını sağlar.
* **Kaynak paylaşımı:** Her bileşene yettiği kadar kaynak sağlar (örneğin Klavye girişi için sadece birkaç KB)
* **Güvenlik ve yetkilendirme:** Kullanıcı ve bilinen servislere yetkilendirme yapar ve güvenliğini sağlar. Ayrıca uygulamaların donanıma **doğrudan** erişimini kısıtlayarak sistemi korur.

## Örnek Kernel Mimarileri
* **Monolitik (monolithic):** Bütün servisleri tek bir yapı altında yüksek yetki ile çalıştırır. Çok hızlıdır ama tek bir hata çökmesine neden olur. (Örneğin: Linux, UNIX)
* **Mikro çekirdek (microkernel):** Çekirdek içinde **sadece** en temel işlevler bulunur. Sürücüler ve servisler userspace'de çalışır. Güvenlidir fakat yavaştır. (Örneğin: Minix, L4)
* **Hibrit çekirdek (hybrid kernel):** Monolitik ve Mikro kernel'in en iyi yanlarını birleştirir. Bazı servisler çekirdekte bazıları userspace'de kalır. (Örneğin, Windows NT, XNU)
