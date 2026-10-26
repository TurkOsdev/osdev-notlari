# QEMU ile emülasyon

QEMU, bir bilgisayarı yazılım içinde taklit eden açık kaynaklı bir emülatördür. Kernel'i gerçek bir makineye yüklemeden, birkaç saniye içinde başlatıp denemeyi sağlar. Hata olduğunda makine çökmez, yalnızca QEMU penceresi kapanır. Bu yüzden OSDev'de kernel'in büyük kısmı QEMU üzerinde geliştirilir.

## Ön koşullar

- [Geliştirme ortamı kurulumu](gelistirme-ortami.md)
- [Build sistemi](build-sistemi.md)

## Nasıl kullanılır

x86-64 için kullanılan program `qemu-system-x86_64`'tür. En basit kullanım, bir ISO imajından açmaktır:

```sh
qemu-system-x86_64 -cdrom os.iso
```

Günlük kullanımda birkaç seçenek daha eklenir:

```sh
qemu-system-x86_64 \
    -M q35 \
    -m 512M \
    -cdrom os.iso \
    -serial stdio \
    -no-reboot \
    -no-shutdown
```

| Seçenek | Ne işe yarar |
| --- | --- |
| `-M q35` | Daha yeni bir PC modeli seçer. Varsayılan model `pc`'dir (i440FX). |
| `-m 512M` | Makineye 512 MiB bellek verir |
| `-serial stdio` | Seri port çıktısını terminale yönlendirir |
| `-no-reboot` | Triple fault olduğunda yeniden başlatmak yerine QEMU'yu kapatır |
| `-no-shutdown` | Makine kapandığında QEMU'dan çıkmaz, yalnızca emülasyonu durdurur |

### UEFI ile açmak

QEMU varsayılan olarak BIOS ile açılır. UEFI için OVMF denen firmware kullanılır. Dosyanın yeri dağıtıma göre değişir:

```sh
qemu-system-x86_64 -bios /usr/share/ovmf/OVMF.fd -cdrom os.iso
```

Bazı dağıtımlar OVMF'yi `OVMF_CODE.fd` ve `OVMF_VARS.fd` olarak iki parça halinde verir. Bu durumda `-drive if=pflash` seçenekleri kullanılır.

### Hata ayıklama seçenekleri

Kernel'in neden çöktüğünü anlamak için QEMU'nun log seçenekleri kullanılabilir:

```sh
qemu-system-x86_64 -cdrom os.iso -no-reboot -d int,cpu_reset -D qemu.log
```

`-d int`, gelen her interrupt ve exception'ı register değerleriyle birlikte yazar. `-D` bu çıktıyı bir dosyaya yönlendirir. Triple fault'tan önceki son exception bu log'da görülebilir.

GDB ile bağlanmak için `-s -S` kullanılır. `-s`, 1234 numaralı port'ta bir GDB sunucusu açar. `-S`, işlemciyi başlangıçta durdurur ve GDB'den devam komutu bekler.

QEMU monitörü ile register'lara ve belleğe bakılabilir. Grafik pencerede `Ctrl+Alt+2` monitöre geçer. `info registers` komutu o anki register değerlerini gösterir.

### KVM

Linux'ta KVM kullanmak QEMU'yu çok hızlandırır:

```sh
qemu-system-x86_64 -accel kvm -cpu host -cdrom os.iso
```

KVM açıkken kod gerçek işlemcide çalışır. Bu yüzden `-d int` gibi bazı log seçenekleri çalışmaz. Hata ayıklarken KVM kapatılmalıdır.

## Sık yapılan hatalar

- `-no-reboot` kullanmamak. Triple fault olduğunda makine sürekli yeniden başlar ve hata görünmez.
- QEMU'da çalışan kernel'in gerçek donanımda da çalışacağını varsaymak. QEMU bazı hatalara karşı daha hoşgörülüdür.
- `-d int` çıktısını KVM açıkken aramak.
- `-serial stdio` ile `-monitor stdio`'yu birlikte kullanmak. İkisi aynı terminali paylaşamaz.

## İlgili notlar

- [Seri port (UART)](../07-drivers/uart.md)
- [Limine boot protokolü](../02-boot/limine.md)

## Kaynaklar

- QEMU dokümantasyonu, "Invocation"; <https://www.qemu.org/docs/master/system/invocation.html> (erişim: 22 Ekim 2026)
- QEMU dokümantasyonu, "GDB usage"; <https://www.qemu.org/docs/master/system/gdb.html> (erişim: 16 Ekim 2026)
- QEMU dokümantasyonu, "QEMU Monitor"; <https://www.qemu.org/docs/master/system/monitor.html> (erişim: 13 Ekim 2026)
- OSDev Wiki, "QEMU"; <https://wiki.osdev.org/QEMU> (erişim: 1 Ekim 2026)
