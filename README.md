# VSS-ve-vPC-Calisma-Notlarim

### VIRTUAL SWITCHING SYSTEM (VSS)

VSS teknolojisi, iki anahtarın son kullanıcıya tek bir anahtar gibi görünmesini sağlar. Tek control plane ile dual active forwarding plane sağlar. Yapılandırmayı basitleştirerek operasyonel karmaşıklıkları ortadan kaldırır. STP döngülerini ortadan kaldırıp iletim kapasitesini arttırır.

VSS konfigurasyonu yapılmadan önce her iki cihazın IOS versiyonlarına bakılır ve her iki versiyonun da aynı olduğundan emin olunur. İki versiyonun da aynı olmaması durumunda hazırlanan konfigurasyon çalışmayacaktır. Kullandığımız her iki anahtarın software ve hardware modellerinin VSS yapısını desteklediğini “show module” diyerek görülür.

VSS konfigürasyonu aşağıdaki gibidir:

    SW1-VSS(config)#switch virtual domain 1
    SW1-VSS(config-vs-domain)#switch 1
    SW1-VSS(config-vs-domain)#switch 1 priority 110
    SW1-VSS(config-vs-domain)#switch 2 priority 100
    SW1-VSS(config)#interface port-channel 1
    SW1-VSS(config-if)#no sh
    SW1-VSS(config-if)#switch virtual link 1
    SW1-VSS(config-if)#exit
    SW1-VSS(config)#int range ten 1/4 – 5
    SW1-VSS(config-if-range)#channel-group 1 mode on
    SW1-VSS(config-if-range)#no sh
    SW1-VSS#switch convert mode virtual
    ----------------------------------------------------------------
    SW2-VSS(config)#switch virtual domain 1
    SW2-VSS(config-vs-domain)#switch 2
    SW2-VSS(config-vs-domain)#switch 1 priority 110
    SW2-VSS(config-vs-domain)#switch 2 priority 100
    SW2-VSS(config)#interface port-channel 2
    SW2-VSS(config-if)#no sh
    SW2-VSS(config-if)#switch virtual link 2
    SW2-VSS(config-if)#exit
    SW2-VSS(config)#int range ten 1/4 – 5
    SW2-VSS(config-if-range)#channel-group 2 mode on
    SW2-VSS(config-if-range)#no sh
    SW2-VSS#switch convert mode virtual
    
VSS konfigürasyonunda ilk adım olarak Virtual Switch Domain ID, 1 ile 255 arasında grup numarası olarak verilir. Sonraki adım cihazlarda aktif-pasif seçimi için priority değeri atamaktır.. Bu durumda Switch1’in priority değeri daha yüksek olduğu için SW1-VSS aktif olur. SW2-VSS ise beklemede yani standby durumunda kalır. VSS teknolojisinde cihazlar arasında konfigurasyon değişikliği ve durum bilgisi güncellemeleri virtual switch link üzerinden yapılır. Konfigurasyonlarda iki anahtarın birbirleriyle bağlantısı için etherchannel yapılmıştır. Ayrıca her bir anahtar için etherchannel’lar “virtual switch link” olarak belirlenmiştir. Daha sonra her iki switch için “switch convert mode virtual” diyerek her iki switchin konfigürasyonları tek bir konfigürasyon haline gelir. İnterface numaraları slot/port düzeninden switch-numarası/slot/port olarak değişir.


### VIRTUAL PORT CHANNEL (VPC)

Virtual Port Channel, diğer bir deyişle multichassis EtherChannel (MEC) olarak bilinen, Cisco Nexus switchlerde vPC eşleri arasında Port-Channel yapılandırma yeteneği sağlayan bir özelliktir.

vPC, Virtual Switching System (VSS) benzerdir. Ancak, vPC ve VSS arasındaki temel fark, VSS'nin tek bir mantıksal anahtar oluşturmasıdır. Bu, hem yönetim hem de konfigürasyon amaçları için tek bir control plane ile sonuçlanır.

vPC konfigürasyonu aşağıdaki gibidir:

    feature vpc
    feature lacp
    vpc domain 1
    !
    vlan 23
    name keepalive
    vrf context keepalive
    interface Vlan23
    vrf member keepalive
    ip address 192.168.1.1/24
    interface Ethernet1/32
    switchport access vlan 23
    !
    vpc domain 1
    peer-keepalive destination 192.168.1.2 source 192.168.1.1 vrf keepalive
    !
    interface ethernet 1/2-3
    description *** VPC PEER LINKS ***
    channel-group 23 mode active
    !
    vlan 10
    interface port-channel 23
    description *** VPC PEER LINKS ***
    switchport mode trunk
    switchport trunk allowed vlan 10
    spanning-tree port type network
    !
    interface Ethernet1/1
    description *** downstream ***
    switchport access vlan 10
    channel-group 10
    !
    interface port-channel10
    switchport access vlan 10
    vpc 10
    !
    -------------------------------------------------------
    feature vpc
    feature lacp
    vpc domain 1
    !
    vlan 23
    name keepalive
    vrf context keepalive
    interface Vlan23
    vrf member keepalive
    ip address 192.168.1.2/24
    interface Ethernet1/32
    switchport access vlan 23
    !
    vpc domain 1
    peer-keepalive destination 192.168.1.1 source 192.168.1.2 vrf keepalive
    !
    interface ethernet 1/2-3
    description *** VPC PEER LINKS ***
    channel-group 23 mode active
    !
    vlan 10
    interface port-channel 23
    description *** VPC PEER LINKS ***
    switchport mode trunk
    switchport trunk allowed vlan 10
    spanning-tree port type network
    !
    interface Ethernet1/1
    description *** downstream ***
    switchport access vlan 10
    channel-group 10
    !
    interface port-channel10
    switchport access vlan 10
    vpc 10
    !
    
Nexus switchlerde default olarak bütün feature’lar kapalı gelir, bu yüzden ilk başta gerekli olan feature’lar aktif edildi (vPC, lacp). Daha sonra vPC için domain oluşturuldu. “show vpc role” komutunu kullanarak durumları gözetlenebilir. Her iki switchde de keepalive için vlan ve vrf tanımları yapılıp ilgili konfigürasyonlar gerçekleştirildi. vPC domain altında vPC peer keepalive link için destination ve source ip adresleri verildi. “show vpc peer-keepalive” komutu ile bu durumu gözlemlenebilir. Daha sonrasında ise vPC peer link tanımlaması için gerekli konfigürasyonlar yapıldı. “show vpc” diyerek vPC link durumu detaylı bir şekilde görüntülenebilir. En son kısımda ise downstream için vPC konfigürasyonu yapıldı.

### StackWise, VSS ve vPC
Tek bir anahtar hem StackWise hem VSS hem de vPC desteklemez. Bir anahtar yalnızca birini desteklemektedir. Sadece higher-end anahtarlarda bu durum gerçersizdir, yani birden fazla bu teknolojilerden içerebilir. Aşağıda bazı cihazların desteklediği teknolojiler verilmiştir.

StackWise: Catalyst 3750, 3850 and 9000 families
VSS: Catalyst 4500 and 6500 families
vPC: Nexus switch family

##### _Teknojilerin Farklılıklar ve Benzerlikleri_

_Stacking vs VSS_
- Stackingde özel bir kablo kullanılırken VSS için 10G interfaceleri kullanılır.
- Stacking 8-9 memberı destekler, VSS max 2 member içerir.
- İkisinde de tek control plane vardır.
- Access layer switchlerde stacking yapılırken core veya aggregation layer switchlerde VSS yapılır.
- Stackingde detaylı bir konfigürasyon yokken VSS için vardır.
- Stackingde management ve kontrol için dedicated edilmiş bir link bulunmazken VSS’de VSL dediğimiz link yapılandırılır.

_VSS vs vPC_
- vPC nexus switchlere özeldir. VSS ise catalyst 6500,4500 series switchlere özeldir.
- vPCde control plane iki switch için de ayrılmıştır. VSS’de ise mantıksal tek bir switch oluşur ve bir tane control plane vardır.
- vPCde HSRP gereklidir. VSS’de gerekli değildir.
- VPCde her switch management ve konfigürasyonu için ayrı IP vardır. VSSde tek IP vardır.
- vPC Layer2 portchannel destekler. VSS Layer3 portchannel destekler.
- vPCde kontrol mesajları peer link ve peer keepalive link üzerinden taşınır. VSS’de ise VSL üzerinden akış sağlanır.
