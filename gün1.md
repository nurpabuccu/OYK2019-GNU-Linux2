## Dummy Interface

Eğer web servisi kendini WiFi arayüzünün değil Ethernet arayüzünün IP adresine bağlarsa ve Ethernetin bağlantısı kesilirse, Ethernet linkinin ‘DOWN’ durumunda olmasından dolayı web arayüzü erişilemez hale gelir.

Dummy bu sorunu fiziksel arayüze bağlı olmayan, IP adresi atayabileceğiniz sanal bir ‘stub’ ile çözüyor. Böylece her zaman ‘UP’ durumunda olur.

Oluşturulabilecek dummy arayüzünün bir limiti olabilir. Dummy modülünün ‘numdummies’ parametresi ile, grub ile kernel komut satırından  /etc/modprobe.d (yeni) veya /etc/modprobe.conf (eski) dosyalarından ‘ options dummy numdummies=4 ’ satırını ekleyerek bu limiti ayarlayabilirsiniz.

Sistemdeki aktif kernel modüllerini görmek için

`$ lsmod`

Aktif bir dummy modülü olup olmadığını görmek için

`$ lsmod | grep dummy`
komutlarını kullanabilirsiniz

Dummy modülünü yüklemek için

`$ modprobe dummy`

Ağ aygıtlarını görüntülemek için 

`$ ip link`

Bir dummy ağ aygıtı oluşturalım ve bir ip adresi verelim

`$ ip link add dummy0 type dummy`

`$ ip adar add 10.0.5.5/24 dev dummy0`
//silmek için del parametresi kullanabilirsiniz

dummy0 ağ arayüzünü up durumuna getirmek için

`$ ip link set dummy0 up`

Yapılan işlemleri 
`$ ip a`
komutu ile görebilirsiniz

dummy0 ekledik fakat bir network ile iletişimi yok şu an. Bunun için virtual ethernet device(veth) eklenir.

## Veth

Veth cihazı local Ethernet tünelidir. Çiftler halinde oluşturulur. İki cihaz arasındaki bağ hiçbir şekilde koparılamaz. Bir cihaza gelen paketler direkt olarak diğer cihazadan da görülebiliyor. Cihazlardan biri down olduğunda bağ durumuda down dır.

Veth çifti oluşturmak için

`$ ip link add veth0 type vets peer name veth1.`
//peer vermesekte default olarak bir çift oluşuyor

`$ ip link set vet0 up`

Bir ağ kartına gelen paketlerin başlıklarını veya içeriklerini görüntülemek için paket analiz programı tcpdump kullanılabilir.

## Bridge

Bridge bir network switch gibi davranır. Kendisine bağlı olan arayüzler arasında paketleri yönlendirir. Genellikle routerlarda, gatewaylerde veya sanal makineler ve bir hosttaki network namespaceler arasında  paket yönlendirmede kullanılır.

Bridge aygıtlarını kullanmak için bridge-utils paketi indirilir.

Bridge oluşturmak için

```
$ brctl addır bridge0
$ brctl show
$brctl addif bridge0 dummy0.   //addif = add interface
```

Ağ arayüz konfigürasyonunu /etc/network/interfaces  dosyasında yapabilirsiniz.

## Cgroups (Kontol Grupları)

Croups processlerin kullandıkları kaynakları sınırlandırmaya ve izole etmeye yarayan Linux kernelinin bir özelliğidir. Hiyerarşik bir yapıya sahiptir. Alt gruplar bağlı olduğu grubun niteliğini taşırlar. Cgroups altında birden fazla hiyerarşi yer alabilir. 8 tane kaynak kontrolcüsü içerir.

/sys/fs/cgroup/

blkio — Blok aygıtlarına olan erişimi kontrol eder

cpu — Processin kullanabileceği cpu miktarı ya da cpularla ilgili sınırlama getirilebiliyor

cpuacct — Processin cpu kullanımını otomatik olarak rapor eder

cpuset — Processin fiziksel olarak hangi işlemcide çalışacağı kontrol edilir

devices — Processlerin aygıtlara olan erişimi kontrol eder

freezer — Processlerin çalışmalarıyla ve durdurmalarıyla ilgili kontrolleri yapmayı sağlar

Memory —  Processlerin kullanabilecekleri ram ve swap miktarını kısıtlayabilmeyi veya görüntülemeyi sağlar

net_cls — cgroups’un ağ aygıtının kullanımına dair sınırlandırmaları yapar

`$cgcreate-g cpuset:/deneme`  
—> cpuset dizininin altında deneme isimli bir grup oluşturur, aksi belirtimediği takdirde hiyerarşik olarak tanımlanır, her bir dizinin ayrı hiyerarşisi vardır

`$ cat /sys/fs/cgroup/cpuset/cpuset.cpus`
—> Cpu sayısı

`$ cat /proc/cpuinfo`
—> Cpu hakkında bilgi

deneme grubu için cpu ayarı yapacağız
`$ echo “0” > /sys/fs/cgroup/cpuset/deneme/cpuset.cpus`
—> grubun kullanabileceği cpu sayısı verildi, 0 = 1 cpu

`$ echo “0” > /sys/fs/cgroup/cpuset/deneme/cpuset.mems`
—> grubun kullanabileceği memory alanı girilir, her cpu’nun kendi memory alanı vardır

`$ echo $$ > /sys/fs/cgroup/cpuset/deneme/tasks`
—> terminalin pid si tasks e girildiğinde  bu terminalden çalıştırdığım komutların hepsi deneme grubunun altında da çalışır

`$ cgcreate -g memory:/deneme`
`$ stress -m 1 --vm-bytes 1g --vm-hang 0`
—> stress programıyla 1gb memory kullacak bir process oluşturduk

`$ echo $((512*1024*1024)) > /sys/fs/cgroup/memory/deneme/memory.limit_in_bytes`
—> deneme grubu içinde 512 MB’tan büyük process çalıştırılamayacak

`$ cat /sys/fs/cgroup/freezer/deneme/freezer.state`
—> THAWED = Process cpu’da fırsat bulursa çalışır
FROZEN = Process durur

## Network Namespaces (NetNS)
Network kaynaklarını birbirinden izole etmek için kullanılır.

`$ ip netns add netns01`
—> netns01 adında bir namespace oluşturduk

`$ ip netns list`
—> mevcut NetNS listesi

`$ ip link set veth1 netns netns01`
—> veth1 netns01 içine eklendi

`$ ip netns exec netns01 ip link`
—> sadece netns01 içindeki likleri listelemek için

Bir aygıtın napespace’ini değiştirilen aygıt DOWN durumuna geçer.

Namespace silindiğinde kapsadığı aygıtlarda silinir. Silinmemesini istediğimiz aygıtları öncesinde default namespace’e taşımalıyız.

## Bindmount
Bir dizini birden fazla yerden mount etmek için kullanılır.

```
$ mount --bind /dizin1 /dizin3
$ mount -o bind /dizin1 /dizin3
$ mount -o bind, ro /dizin1 /dizin3   //ro=read only dizine ro veya rw olarak parametre verebiliriz
```
dizin2 dizin3’e mount edilirse dizin3 dizin2’nin içini gösterir. Son mount işlemi esas alınır

dizin2 dizin3’ten umount edilirse dizin3 dizin1’i gösterir.

## Sanallaştırma (Virtualization)

Sanallaştırma, bilgisayar sisteminin sanal bir örneğini asıl donanımdan soyutlanan bir katmanda çalıştıran bir işlemdir. Genel olarak bir bilgisayarda birden fazla işletim sistemi çalıştırmak gibidir.
### Donanım Sanallaştırması (Hardware Assisted Virtualization)

Donanım destekli sanallaştırma, sanal makineleri oluşturan ve yöneten yazılımı desteklemek için bilgisayarın fiziksel bileşenlerinin kullanılmasıdır. Yazılım ve donanım arasında fiziksel kaynağı kontrol eden bir sanallaştırma yöneticisi hypervisor veya virtual machine monitor(VMM) vardır.

#### Tip-1/Bare Metal Hypervisor

Tip-1 hipervizörler direk donanımın üstünde çalışan bir yazılım katmanıdır. Bütün donanımı hipervizçr kontrol eder. 

Tip-1 hipervizörün kendisi bir uygulamadır.

Örnek: VMWare ESXi, Microsoft Hyper-V, Oracle VM …

#### Tip-2 Hosted Hypervisor

Tip-2 hipervizörler ile donanım arasında OS katmanı bulunur. Donanımı OS kontrol eder yani hipervizör donanıma ulaşmak için işletim sistemini kullanmalıdır. Tip-2 hipervizör OS katmanlarından dolayı Tip-1 hipervizörden daha yavaştır diyebiliriz.

Örnek: VirtualBox, VMWare Workstation, Linux Kernel …

## Process Virtualization

## Linux Container

LXC, tek bir Linux çekirdeği kullanarak bir kontrol ana bilgisayarında birden çok yalıtılmış Linux sistemini çalıştırmak için işletim sistemi düzeyinde bir sanallaştırma yöntemidir.

`$ apt install lxc`

/etc/default/lxc

`$ systemctl status lxc-net `
—> Servisin durumunu gösterir

`$ nano /etc/default/lxc-net`
—> Lxc network konfigürasyonunu yapacağız

```
USE_LXC_BRIDGE="true"
LXC_BRIDGE="lxcbr0"
LXC_ADDR="10.0.10.1"
LXC_NETMASK="255.255.255.0"
LXC_NETWORK="10.0.10.0/24"
LXC_DHCP_RANGE=“10.0.10.2,10.0.10.254"
```

`$ systemctl start lxc-net`
—> Servisi başlatır

/etc/default/lxc-net dosyasında değişiklik yapıldığında servis restart edilmelidir.

Bridge cihazlarını görmek için

`$ sudo brctl show`

NAT ile bağlı bridge’deki cihazlar internete çıkabilir

`$ iptables -vn -L -t nat`
—> NAT tablosunun bir listesini getirir

Dhcp’nin çalışıp çalışmadığını kontrol etmek için

`$ ps fuaxwww | grep dnsmasq`

`$ lxc-checkconfig`
—>  Lxc izinlerini gösterir

`$ nano /etc/lxc/default.conf`
—> Oluşturulan her lxc için ortak ayarları burada giriyoruz

`$lxc-create --name debian01 -t download`
—> lxc container oluşturuyorum debian01 adında. Kurabileceğim imajlar ekranda listelenir sırayla distribution, release, architecture seçilir

`$ lxc-ls`
—> Kurulan containerları listeler

`$ lxc-start --name debian01`
—> Container başlatılır

`$ lxc-ls —fancy`
—> Container durumunu gösterir

`$ ps fuaxwww`
—> Container processlerini buradan görebiliriz

/var/lib/lxc —> Kurulan lxc containerlar bu dizinde bulunur

/var/lib/lxc/lxc01/rootfs —> Container kök dizini

`$ lxc-attach --name lxc01`
—> Çalışan container içinde işlem yapmamızı sağlar, lxc01 terminali açılır

`$ lxc-attach --name lxc01 --touch dosya.txt`
—> Container terminaline girmeden bash üzerinde komutu çalıştırır

Container oluştururken komut satırından indireceğimiz imajı parametre olarak verebiliriz

`$ lxc-create --name centos -t download -- -d centos -r 7 -a amd64`

Lxc oluşturduğumuzda /var/cache/lxxc/download oluşturuluyor. Tekrar bir indirme yaparken cache’ten alınıyor. Container destroy edilirse cache siliniyor.

/var/lib/lxc/lxc01/config —> Container konfigürasyon ayarları

**Daha fazla bilgi için**

man lxc.container.conf = https://linuxcontainers.org/lxc/manpages//man5/lxc.container.conf.5.html

https://wiki.archlinux.org/index.php/Linux_Containers

https://wiki.debian.org/LXC
