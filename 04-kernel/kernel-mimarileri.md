# Kernel (Çekirdek) Mimarisi Nedir?

**Kernel (çekirdek) mimarisi**, işletim sistemi çekirdeğinin hangi bileşenlerden oluşacağını, bu bileşenlerin birbirleriyle nasıl iletişim kuracağını ve hangi görevlerin kernel space veya userspace içerisinde çalışacağını belirleyen tasarım yaklaşımıdır.

Kernel, uygulamalar ile bilgisayar donanımı arasında temel bir katman görevi görür ve sistem kaynaklarını yönetir.

## Temel Görevleri

* **Donanım yönetimi:** CPU, RAM, depolama aygıtları, klavye, mouse ve diğer donanımların işletim sistemi tarafından kullanılmasını sağlar.
* **Kaynak yönetimi:** CPU zamanı, bellek ve diğer sistem kaynaklarının çalışan süreçler arasında paylaştırılmasını sağlar.
* **Güvenlik ve yetkilendirme:** Süreçlerin hangi kaynaklara erişebileceğini kontrol eder ve uygulamaların donanıma doğrudan erişimini sınırlar.
* **Süreç yönetimi:** Programların çalıştırılması, zamanlanması ve birbirleriyle iletişim kurması gibi işlemleri yönetir.

## Kernel Mimarileri

* **Monolitik çekirdek (Monolithic Kernel):** Sürücüler, dosya sistemleri, ağ yığını ve birçok sistem servisi kernel space içerisinde çalışır. Bileşenler doğrudan iletişim kurabildiği için yüksek performans sağlayabilir. Ancak kernel içerisindeki kritik bir hata tüm sistemin çökmesine neden olabilir. </br> **Örnek:** Linux, FreeBSD.

* **Mikro çekirdek (Microkernel):** Kernel içerisinde yalnızca süreç zamanlama, bellek yönetiminin temel bölümleri ve IPC gibi kritik işlevler bulunur. Sürücüler ve birçok sistem servisi userspace içerisinde çalışır. Bu yapı hata izolasyonunu ve güvenliği artırabilir ancak bileşenler arasındaki mesajlaşma ek performans maliyeti oluşturabilir.</br> **Örnek:** MINIX 3, QNX, L4 ailesi.

* **Hibrit çekirdek (Hybrid Kernel):** Monolitik ve mikro çekirdek yaklaşımlarından özellikler birleştirir. Mikro çekirdek benzeri bir tasarım kullanırken performans amacıyla bazı servisleri kernel space içerisinde çalıştırabilir. </br> **Örnek:** Windows NT, Apple XNU.
