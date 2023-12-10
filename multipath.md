Multipath Quic
================

A QUIC szabvány bevezeti a Connection ID fogalmat, ami az IP cím és a port mellett, a végpontok azonosítani tudjdák egymást, hogy az adott streamhez milyen csomagok tartoznak. Ezért ha változik az IP címe egy végpontnak (mozgó végpontok esetén gyakori), továbbá is ismerni fogja a két végpont egymást, és melyik csomagok tartoztak melyik stream-hez.

Elökészítés
-----------

Első sorban, tényleges hálózat konfigurációja elég nagy igényekkel jár, különbötő számítógépek kell rendelkezésre áljanak. Egy ennél sokkal egyszerübb megolás egy emulált virtuális hálózat létrehozása, amelyik közel ugyanúgy viselkedik mint egy fizikai hálózat. Erre ad lehetőséget a [Mininet](https://mininet.org/) alkalmazás, aminek segítségével hálózati kommunikációt emulál.

*Opcionális*: A Mininet Python utasításokat használ ezért Python3 segítségével könnyeben tudunk Mininet parancsokat kiadni, illetve host-okat konfigurálni (Linux környezetben már le van töltve).

Továbbá szükség lessz egy alkalmazásra ami lehetőve teszi a hálozati forgalom és a kommunikáció csomagjainak viszgálatát. Erre alkalmas a Wireshark, ráadásul a Mininet interface-ei láthatóak lesznek számára. Ha mégsem müködne, akkor a [Mininet Walkthrough](https://mininet.org/walkthrough/) segítséget nyújt arra, hogy a Wireshark felismerje az általa definiált interface-eket.

Ellenörizzük, hogy ha helyessen jártunk el. Az első a Wiresharkot indítja el a háttérben, a második a Mininetet. A Minineten indítsunk el az alapértelmezett topológiát. Ekkor meg kell jelenjen Wiresharkban a virtuális hálózat interface-ei.

```bash
sudo wireshark &
sudo mn
```

Indítsuk el a csomagelkapásokat mind a két inteface-én az alapértelmezett topológiában (lást [WSH dokumentáció](https://www.wireshark.org/docs/wsug_html_chunked/)). Adjuk ki a **pingall** parancsot mininet termináljában. Ekkor látnunk kell *echo ping* üzeneteket a Wireshark-ban.

A multipath teszteléséhez nem lessz elég az alapértelmezett topológia, sajátot kell létrehozni. A mininet/custom könyvtárban hozzunk létre egy úgy *Python* program file-t, ez lessz az új topológia (az útmutatóban ennek a neve **picoquic_2sw_2host_test.py**). Ebbe a következő kódot másodljuk be.

```python
import os
from mininet.topo import Topo
from mininet.net import Mininet
from mininet.node import Node
from mininet.log import setLogLevel, info
from mininet.cli import CLI
from mininet.link import Intf
from mininet.node import Controller

class PicoquicNetworkTopo( Topo ):
    #Builds from network topology
    def build(self):
        #Adding switches
        s1 = self.addSwitch('s1')
        s2 = self.addSwitch('s2')

        #Adding hosts
        h1 = self.addHost('h1', ip='192.168.0.1/24')
        h2 = self.addHost('h2', ip='192.168.0.2/24')

        #Connecting hosts and switches
        self.addLink(s1, h1)
        self.addLink(s1, h2)
        self.addLink(s2, h1)
        self.addLink(s2, h2)

def run():
    #Bootstrap a Mininet network using the topology
    mytopo = PicoquicNetworkTopo()
    net = Mininet(topo=mytopo)

    #Run a configuration in this topology
    net.start()

    #Run multipath configuration
    net['h1'].cmd('ip addr add 192.168.1.1/24 dev h1-eth1')
    net['h1'].cmd('ip link set h1-eth1 up')
    net['h2'].cmd('ip addr add 192.168.1.2/24 dev h2-eth1')
    net['h2'].cmd('ip link set h2-eth1 up')

    CLI(net)
    net.stop()

if __name__ == '__main__':
    #This is the main
    setLogLevel('info')
    run()

topos = {'mytopo': PicoquicNetworkTopo }
```

A kódban láthatóan hivatkozunk egy bash file-ra. Ebben van definiálva a hostoknak a másodlagos interface-ük. Az egyik ilyen bash file tartalma a következő (*config_h1.sh*):

```bash
#Save connection keys to file for decryption
ip addr add 192.168.1.1/24 dev eth1
ip link set eth1 up
```

Hasonlóan configuráljuk a **config_h2.sh** file-t is.

Picoquic
--------

A Picoquic implementációt a következő linken lehet elérni: https://github.com/private-octopus/picoquic <br> Itt található egy útmutató is, hogy hogy kell letölteni, illetve lefordítani a programot, illetve milyen további komponensekre van szükség.

A lemásolt repository-ban navigáljunk a picoquic könyvtárba. Ebben található egy **picoquicdemo**, amit lefordítva, ennek parancsait alkalmazni tudjuk ebből a könyvtárból. 

### Egyszerű kliens-szerver kommunikáció

Egy egyszerü kliens-szerver kapcsolatot akarunk létrehozni, majd átküldei egy csomagot a hálózaton. Látható lessz a handshake protokolra példa, illetve szemlélteti a Connection ID (CID) használatát is. 

Ehhez egyenlőre elég lessz a mininet alapértelmezett topológiája, ahhol két host egy switch-en keresztül össze van kötve. Viszont, annak érdekében hogy felismerhetőek maradjanak a végpontok érdemes a mac címet rögziteni, hogy minden futtatáskor ugyanaz maradjon. Elöször indítsunk egy Wireshark-ot a csomagok elkapásához a háttérben, majd a következő parancsal indítsuk el a mininetet is.

```bash 
$ sudo wireshark &
$ sudo mn --mac
```
 Rögtön látható a létrehozott csomópontok, illetve a kapcsolatok létrehozása. Legyen a h1 a szerver csomópont, a h2 pedig a klines. Késöbb szükségünk lesz a szerver IP címére ezért érdemes a mininet segítségével lekérdezni.

```bash
mininet> h1 ifconfig
h1-eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.1  netmask 255.0.0.0  broadcast 10.255.255.255
        inet6 fe80::200:ff:fe00:1  prefixlen 64  scopeid 0x20<link>
        ether 00:00:00:00:00:01  txqueuelen 1000  (Ethernet)
        RX packets 21  bytes 2527 (2.5 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 7  bytes 646 (646.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

 A mininet csak hálózatot emulál, a hostok nem virtuális gépen futnak, hanem egy namespace-be vannak körülkerítve, ami a gazda gépre le van vetítve (pl. ezért látható ugyanaz a command log mind a két host-on). Nyissuk meg a két host xterm-jét:

 ```bash
 > xterm h1 h2
 ```

 A h1 xterm-jén navigáljunk a picoquic állomány gyökerébe. Az első részben lévő utasítások helyes fordítása esetén a következő parancsnak müködnie kell (ha ez nincs így akkor el kell végezni újra a fordítás lépéseit).

 ```console
 # ./picoquicdemo -h
 ```
 A szert a következő utasítással lehet elindítani.

 ```console
 # ./picoquicdemo -1 -w ./htmlroot -n server -p 443
 ```

- <i>./picoquicdemo</i>: A picoquic szolgáltatásait szeretnénk igénybe venni. C kód futtatását idézi elő.
- <i>-1</i>: ha a szerver küldött csomagot akkor kapcsoljon ki.
- <i>-w ./htmlroot</i>: itt kell keresni a szolgáltatott file-okat. Ebbe egy index.html file-t érdemes létrehozni.
- <i>-n server</i>: a picoquic szemantikájában fontos nevet adni a szervernek.
- <i>-p 443</i>: innen tudja a picoquic hogy szerver, illetve milyen porton kell hallgason. A QUIC szabványa szerint 443 porton a standard port QUIC forgalomra.

A klinst  következő utasítással lehet elindítani:

```console
# ./picoquicdemo -o ./received -n server 192.168.0.1 443 index.htlm
```

- <i>-o ./recieved</i>: ide irja a kapott file-t. Mivel mind a két host le van vetítve a gazda gépre, ha nem adnánk meg -w és -o parancsokat akkor ugyaoda irja mint ahhonna olvassa a file-t, ezért nem lehetne leellenőrizni hogy eredeményes volt-e a kapcsolat.
- <i>192.168.0.1 443</i>: ennek a szervernek szeretne olvasási kérést küldeni, az adott porton (az IP cím a h1 IP címe).
- <i>index.html</i>: A letöltendő file-nak a neve

Ha megnézzük a wiresharkban a kapcsolat létrejottét, illetve a csomagok elkapását, észre lehet venni, hogy títkosítva vannak ezek az adatok, nem látható a Communication ID szerinti stream forgalom. Ez abból adódik, hogy a QUIC szabvány szerint, a titkosítás nem a szállítási reteg felett van, hanem bele van ágyazva az alkalmazás retegbe. Ezért a wireshark nem tudja elkapni a titkosítás kulcsait. Ennek következtébe le kell tárolni a generált kulcsokat, és kézzel meg kell adni ezeket (segítség ehhez a folyamathoz: https://wiki.wireshark.org/TLS). Ezért a szerver parancsát módosítani kell a következőképpen:

```console
# SSLKEYLOGFILE=\"sslkeys.log\" ./picoquicdemo -1 -w ./htmlroot -n server -p 443
```

Ha a kliens oldalon a kilépés 0 arument értékkel lép ki akkor a kapcsolat sikeres volt. Ez ellenörizhető azzal is, hogy rendelkezésre áll a html file a received mappában.

### Kliens-szerver kommunikácio multipath segítségével

A mininet alapértelmezett topologiája már nem lessz alkalmas a multipath mechanizmusainak tesztelésére, ezért sajátot kell defininiálni. Egy példa topológia megtalálhato a ~/mininet/custom/picoquic_test könyvtáron belül. Az itteni topológia két host-ból és két switch-ből áll ahol minden switch minden hostal össze van kötve. Továbbá megtekinthető az is, hogy a topológia definiálásán belül, a host-ok IP cimet kaptak.

A QUIC multipath lehetővé teszi hogy több úton vegyen részt a kommunikáció a két végpont között. Ez csak akkor lehetséges ha a végpontok több interface-el rendelkeznek. Ekkor a a routerek egy másik útvonalon továbbitják a forgalmat (mivel az interface-ek különböző IP címmel rendelkeznek). Ennek müködése szerint a főkapcsolat kialakulása után, a handshake során a két fél egyezteti milyen multipath képeségekkel rendelkeznek, majd ezek útan a cliens fél átad egy <i>alternativ IP címet</i>, amivel a szerver kezdeményezhet kapcsolatot. 
(további információ elérhető a <a href="https://datatracker.ietf.org/doc/draft-ietf-quic-multipath/">QUIC Multipath Draftban</a>)

 Ennek következtében a topológia definiálása során shell parancsok segítségével bővitettük a két host-ot új interfacekkel, így a két host-hoz tartozó IP címek:
 - <b>h1:</b> 192.168.0.1/0 és 192.168.1.1/1 ahol a /x jelenti az interface indexét
 - <b>h1:</b> 192.168.0.2/0 és 192.168.1.2/1 ahol a /x jelenti az interface indexét

 Az előre definiált topológia használatát az alábbi parancs mutatja be:

 ```bash
 $ sudo mn --custom ~/mininet/custom/picoquic_test/picoquic_2sw_2host_test.py --topo=mytopo
 ```
A topológia Python segítségével van definiálva ahogy a <a href="https://github.com/mininet/mininet/wiki/Introduction-to-Mininet">Mininet Python API</a> segítségével, ezért Pythonnal közvetlen futtatni lehet a mininet kódot. Akár a Minineten belül lehet futtatni python kódot a host-okon (Python 2-esben a http.server neve SimpleHTTPServer)

```bash
$ sudo python3 /home/cloud/mininet/custom/picoquic_test/picoquic_2sw_2host_test.py
```

A szerver oldalt a következő képpen kell felparaméterezni. A -M a multipath típusát adja meg ami lehet none(0) full(1), simple(2) vagy both(3).

```console
# SSLKEYLOGFILE=\"sslkeys.log\" ./picoquicdemo -1 -M 2 -w ./htmlroot -n server -p 443
```
A kliensnél figyelni kell, hogy adjunk meg alternativ elérhetőséget is. Ezt az -A paraméterrel lehet megtenni.

```bash
./picoquicdemo -M 2 -A 192.168.0.2/1,192.168.1.2/2 -o ./received -n server 192.168.0.1 443 index.html
```

Habár a parancs valóban a multipath implementációt használja fel, ennek nyomát mégsem lehet látni a futtatás során. Ennek oka az, hogy az index.html file túl egyszerű, nincsennek benne több objectumok amik beolvasásához több stream-et kellene nyitni. Helyette a -B paraméterrel meg lehet adni hogy mennyi byte csomag menjen át a hálózaton. A fenti kliens parancs helyett futtasuk le a következő parancsot:

```bash
./picoquicdemo -M 2 -A 192.168.0.2/1,192.168.1.2/2 -o ./received -B 100000 -n server 192.168.0.1 443
```

Most már a Wireshark segítségével látható, hogy a cél és a forrás IP címek bejegyzésben megjelentek az alternatív IP címek, illetve hogy az ezen csatornák CID-jével megegyező NEW_CONNECTION csomag tartalmában, szintém megtalálhatóak.

Quiche
------------

A Quiche a Google Implementácioja a QUIC protokollnak. A legnagyobb külömbség a picoquic-el szemben, hogy a szerver parancsok és a kliens parancsok más szemantikával rendelkeznek: a klines ./quiche-klient, a szervernek pedig ./quiche-server. Így nehezebb összekeverni a két parancs használatát.

### Elökészítés

Quiche implementációrol szoló útmutató alérhető itt: https://github.com/qdeconinck/quiche <br>
Mivel a Quiche Rust-ban íródott, szükség lessz [Rust 1.54](https://rustup.rs/) (vagy későbbi) fordítora. A Github repository-t klónozzuk le, az alább megadott parancsal:

```bash
git clone --recursive https://github.com/qdeconinck/quiche.git -b multipath
```

A fordításhoz szüksége lessz *Cargo* alkalmazásra, mivel a Quiche títkosítást [BoringSSL](https://boringssl.googlesource.com/boringssl/) segítségével valósul meg, amit autómatikusan hozzácsatol a fórdító. A fordítás így néz ki:

 ```bash
 cd quiche
 cargo build
 cargo test
 ```

 Az utolsó parancs futtatja az examples könyvtárban található összes tesztet. Ha ezek sikeresen lefutnak, akkor a megoldás helyes volt.

### Egyszerű kliens-szerver kapcsolat

Elöszőr probáljuk ki azt hogy hogyan néz ki a kapcsolódás a Quiche esetén. Ehhez csak minimálisan szeretnénk felparaméterezni a parancsokat hogy a kapcsolat létrejöjjön. Navigáljunk a quiche/target/debug állományba. Innen tudjuk kiadni a quiche parancsokat. Nézzük meg milyen paramétereket lehet megadni a Quiche-nek

```bash
./quiche-server --help
./quiche-kliens --help
```

Ezek közül az alábbi opciókat paraméterezzük fel:

Paraméter | Érték | Target | Magyarázat |
| - | - | - | - |
| --listen | 0.0.0.0:4433 | server | A szerver ezen interface-en figylje a bejövő QUIC kéréseket. Az értékkel megadjuk hogy minden interface-en figylje az adott portszám forgalmát.
| --cert | /home/cloud/src/bin/cert.pem | server | A TLS hitelesitő file-nak az elérésí útvonala |
| --key | /home/cloud/src/bin/key.pem | server | A TLS hitelesítés során a encryption key-nek az elérési útvonala |
| --trust-origin-ca-pem | /home/cloud/src/bin/cert.pem | client | A TLS hitelesítő file-t a kliensel is egyeztetni kell. Mindig használni kell, amikor server oldalon az implicitnél külömböző hitelesítő file-t akarunk megdani. |
| --index | index.html | server | Az index file-t határozza meg. Ezt kell továbítani ennek a szervernek, ha QUIC kérésé érkezik felé. |
| --root | . | server | Az index file elérésí útvonalát lehet megadni. Az érték jelentése, hogy az index file-t a programkönyvtár gyökerébe keresse. |
| --multipath |  | server/client | A multipath szolgáltatás bekapcsolása. Implicit esetben nincs bekapcsolva. |
| -A | másodlagos IP cím | client | Az elternatív IP címet lehet megadni. Itt ténylegesen csak az alternatív kell. Nem kell index-et is hozzárendeleni. |

Sajnos a letöltött repository-ban a --cert és --key default mappájában hiányoznak a hitelesítéshez szükséges file-ok. Ennek köszönhetően generálni kell ezeket a file-okat és beletenni az alapértelmezett mappába. Egy másik müködőképes lehetőség, hogy felhasználjuk a picoquic TLS hitelesítő adatait. Ha így járunk el, akkor a következőképpen lehet felparaméterezni a parancsot:

```bash
./quiche-server --listen 0.0.0.0:443 --cert /home/cloud/picoquic/certs/cert.pem --key /home/cloud/picoquic/certs/key.pem --index index.html --root .
```

És a kliensnek megfelelő parancsa:

```bash
./quiche-client --trust-origin-ca-pem /home/cloud/picoquic/certs/cert.pem https://192.168.0.1:443/index.html
```

Ha wiresharkban elemezni kívájuk a forgalmat el kell végezni úgyanazokat a lépéseket amiket a picoquic során is elvégeztünk. A szerver oldalon a parancs elé az **SSLKEYLOGFILE="sslkey.log"**

Ha minden helyes volt, akkor látnunk kell a clines terminálban az index.html file kiiársát. A Wireshark segítségével megnézhetjuk a forgalmat, és hogy milyen csomagok érkeztem meg az interface-en.

### Kliens-szerver kapcsolat multipath használatával

A multipath funkció használatához meg kell adni, úgy a kliens mint a szerver oldalon a multipath engedélyezését. Ezt a --multipath szintaktikával tehetjük meg. Bővítsük ezzel ki a kliens és a szerver elínditásához szükséges parancsokat.

```bash
SSLKEYLOGFILE="sslkey.log" ./quiche-server --listen 0.0.0.0:443 --cert /home/cloud/picoquic/certs/cert.pem --key /home/cloud/picoquic/certs/key.pem --index index.html --root . --multipath
```

```bash
./quiche-client --trust-origin-ca-pem /home/cloud/picoquic/certs/cert.pem https://192.168.0.1:443/index.html --multipath -A 192.168.1.1
```

Hogy a mulitpath funkciót kikényszerítsük szükség lessz egy nagyobb index.html file-ra. Egy ilyen file-t lehet letölteni [erről](https://demo.borland.com/testsite/stadyn_largepagewithimages.html) a linkről, vagy használni lehet a quiche/target/debug/examples könyvtárban levő index.html file-t. Ezt viszont már nem szeretnénk hogy kiirja a konzolra ezért a következőképpen kell megváltoztatni a parancsot.