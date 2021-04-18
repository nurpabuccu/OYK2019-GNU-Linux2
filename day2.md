## Docker

Docker, containerlar kullanılarak uygulamaları oluşturmayı, dağıtmayı ve çalıştırmayı sağlayan bir araç. Docker LXC gibi tamamen izole bir sistem sunmak yerine yalnızca çalıştırılacak uygulamayı izole eder. Docker başta LXC tabanlı olarak çalışıyor LXC'nin cgroup çağrılarını kullanırken sonradan kendi özlleştirilmiş cgroup implementasyonunu geliştirdi.

### Docker Daemon (Docker Engine)

Docker daemon arkaplanda sürekli olarak çalışan ve tek ana makinede containerların birbirinden izole bir şekilde çalışmasını ve tüm işlemlerinin yönetilmesini sağlayan işlemdir. Container ise Docker Daemon tarafından Linux çekirdeği içinde oluşturulan izole çalışan process'tir.

### Kurulum

<https://docs.docker.com/engine/install> dökümantasyonunu takip ederek sisteme göre kurulum gerçekleştirebilirsiniz.

Debian tabanlı bir sistem için:

```
$ sudo apt install apt-transport-https ca-certificates curl gnupg2 lsb-release software-properties-common
```

Docker GPG Key'ini ekleme
```
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Docker reposunun apt sources'a eklenmesi
```
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
```
Reponun eklenip eklenmediğini /etc/apt/sources.list'ten kontrol edebilirsiniz.

Paket listesini güncellemek için 
```
$ sudo apt update
```

Docker engine'i yüklemek için
```
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Yüklemenin doğru tamamlandığını kontrol etmek için bir docker container'ı ayağa kaldırabilirsiniz.
```
$ sudo docker run hello-world
```

Bu komutla bir test imajı indirilip container içinde çalıştırılacak, ekrana Hello World' yazdıktan sonra kapanacaktır. <https://hub.docker.com/search?q=&type=image> adresinden başka docker imajları indirebilirsiniz.

Docker container içinde yalnızca bir process çalıştırdığı için process ölürse veya durdurulursa docker container'ı da ölüyor.

`$ sudo docker ps` -> Çalışan container listesi

`$ sudo docker ps -a` -> Tüm container'ların listesi

Docker'da container'a bir isim verilmezse kendi bir isim atıyor.

Docker container çalıştırılıp parametrik olarak içinde bir program çalıştırmak için

`$ sudo docker run -it ubuntu /bin/bash`

    -i : Container'ın std inputunu çalıştırılan terminalin std inputuna bağlıyor 
    -t : Terminal bağlantısını verilen /bin/bash ile yapıyor

`docker stop <container_name>` -> Çalışan container'ı durdurur

`docker start <container_name>` -> Durdurulan container'ı yeniden başlatır

`docker start -a -i <container_name>` -> Container tekrar çalıştırılıp içine girilir

    -a : Output container'a bağlanır
    -i : Input container'a bağlanır

`docker attach <container_name>` -> Çalışan docker container'a attach olur

`docker attach --detach-keys="ctrl-x" <container_name>` -> Container'a attach olurken belirtilen key kombinasyonu ile detach olması belirtilebilir. ctrl-x ile docker container kapatılmadan içinden çıkılabilir.

`docker rm <container_name>` -> Container'ı silmek için

Eğer `docker info` ile storage driver'a bakarsanız default olarak overlay2 olarak geldiğini görebilirsiniz. Örneğin birden fazla container bir ana imajı kullanıyor diyelim. Ana imaj üstünde bir değişiklik yapıldığında container için ayrı bir imaj oluşturulur, değişiklik yapmayan containerlar aynı imajı kullanmaya devam ederler. Böylelikle aynı kök imaj tekrar tekrar kopyalanmayıp bir kez oluşturulup kullanılıyor buda hem hız hem da daha az sistem kaynağı (depolama) kullanılmasını sağlıyor.

## Donanım Sanallaştırma

Önce sisteminizde donanım sanallaştırma desteğinin açık olup olmadığını kontrol edeceğiz.

`$ grep -E "vmx|svm" /proc/cpuinfo` -> Komut çıktı üretiyorsa sanallaştırma etkindir

`$ sudo apt install qemu-kvm` -> Kvm indirilir

Not: Virtualbox'ta KVM çalıştırmak mümkün değil. VMWare kullanabilirsiniz.

KVM için KVM çalıştırmak için `/etc/modprobe.d/kvm-nested.conf` dosyası oluşturulup içine 

    Options kvm_intel nested=1

yazılır. Daha sonra KVM modülü kaldırılıp geri yüklenir.

```
$ sudo modprobe -r kvm_intel
$ sudo modprobe kvm_intel
```

KVM'i kurduğunuz hostta nested virtualization'un desteklendiğini kontrol etmek için

`$ cat /sys/module/kvm_intel/parameters/nested`

Eğer destekleniyorsa Y veya 1, desteklenmiyorsa N veya 0 çıktısını göreceksiniz.

Sanal makinenin disk olarak göreceği bir alana ihtiyacı var. Bunun için bir sanal disk imaju gerekir.

Disk image : Dosya, ama sanal makinelerdeki işletim sistemleri bu dosyayı bir block device (bir disk gibi) gibi kullanabiliyor.

Örneğin 2GB'lık bir imaj oluşturmak için

`$ sudo qemu-img create disk.img 2G` -> Bulunduğunuz dizine oluşturur

`$ du -hs disk.img` -> Dosyanın diskte kapladığı alanı gösterir

Bu imaj sanal makine oluşturulduğunda host makinede direkt 2GB yer kaplayacak. Fakat biz diske yazıldığı kadar yer kaplaması için CoW imaj oluşturabiliriz.

`$ sudo qemu-img create disk.img 2G -f qcow2`

Bir debian iso'su indiriyoruz sanal makine kurabilmek için.

```
$ wget https://cdimage.debian.org/debain-cd/current/amd64/iso-cd/debian-
10.0.0-amd64-netinst.iso`
```

KVM ile indirdiğimiz iso ile oluşturduğumuz disk imajını da vererek, 512mb hafızalı bir sanal makine oluşuturuyoruz.

```
$ sudo -E kvm -m 512 -hda disk.img -cdrom debian-edu-10.0.0-amd64-netinst.iso -boot d -netdev user,id=user.0 -device e1000,netdev=user.0
```

Libvirt, sanallaştırma platformunu (KVM, Xen, VMware ESXi, QEMU, LCX...) yönetmeyi sağlayan açık kaynak API, daemon ve yönetim sistemi aracıdır. Libvirt, API kütüphanesi, daemon (libvirtd) ve bir komut satırı aracı (virsh) içerir. 

`$ sudo apt install libvirt-daemon-system libvirt-clients virtinst`

Libvirt'in kendi yönettiği ağ yapılandırmalarını listelemek için

`$ sudo virsh net-list —all`

`$ sudo virsh net-start default` -> Network başlatılır

`$ sudo virsh net-list --all` - Libvirt'in yönettiği ap yapılandırmalarının listesi

Default network'ün her açılışta otomatik başlatılması için

`$ sudo virsh net-autostart default`

`$ sudo virt-install --name kvm1 --ram 512 path=/var/virtdisks/kvm.img,size=1 --network network:default --accelerate /var/lib/libvirt/images/debian-edu-10.0.0-amd64-netinst.iso` -> Libvirt ile kvm1 isimli sanal makine oluşturuldu

`$ virsh list —all` -> Çalışan sanal makineleri gösterir

`$ virsh destroy kvm1` -> Çalışan makineyi durdurur

Sanal makinelerin içerikleri ile ilgili yönetimi kolaylaştıran bir program da [libguestfs](https://libguestfs.org/)'dir.
