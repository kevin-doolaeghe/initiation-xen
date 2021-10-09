# Installation de Xen Hypervisor sur Debian

## Auteur

### Kevin Doolaeghe

## Installation & préparation de l'environnement Debian

* Passage en mode super-utilisateur à l'aide de la commande `su`

* Ajout du dossier `/sbin` à la variable d’environnement PATH pour utiliser les binaires qu’il contient :
```
export PATH=$PATH:/sbin
```

* Mise à jour du système :
```
apt update && apt upgrade
```

* Installation des paquets ci-dessous :
```
apt install build-essential net-tools bridge-utils ethtool linux-image-amd64 xen-hypervisor-4.14-amd64 xen-utils-4.14 xen-tools iptables-persistant
```
&ensp; &ensp; &rarr;  `build-essential` contient les paquets relatifs à la compilation  
&ensp; &ensp; &rarr;  `net-tools` contient les paquets de base pour la configuration réseau  
&ensp; &ensp; &rarr;  `bridge-utils` permet de créer et gérer les ponts  
&ensp; &ensp; &rarr;  `ethtool` est un utilitaire permettant d'afficher et de modifier certains paramètres de la carte réseau  
&ensp; &ensp; &rarr;  `linux-image-amd64` est l'image de Debian utilisée pour les machines virtuelles  
&ensp; &ensp; &rarr;  `xen-hypervisor-4.14-amd64`, `xen-utils-4.14` et `xen-tools` sont les paquets d'installation de Xen  
&ensp; &ensp; &rarr;  `iptables-persistant` donne accès à la sauvegarde persistante de la configuration `iptables`

### Configuration réseau

* Désactivation de `NetworkManager` :
```
systemctl stop NetworkManager
systemctl disable NetworkManager
```
&ensp; &ensp; &rarr;  La configuration Wifi sera effectuée à l'aide de l'utilitaire `wpa_supplicant`  
&ensp; &ensp; &rarr;  La configuration IP sera effectuée à l'aide du fichier `/etc/network/interfaces`

### 1. Configuration du Wifi :

* Configuration du SSID et du mot de passe du réseau Wifi :
```
wpa_passphrase '<NET_SSID>' <WPA2_KEY> | sudo tee /etc/wpa_supplicant/wpa_supplicant.conf
```
&ensp; &ensp; &rarr;  La configuration du réseau Wifi se trouve donc maintenant dans le fichier `/etc/wpa_supplicant/wpa_supplicant.conf`

* Démarrage automatique de `wpa_supplicant` :
```
cp /lib/systemd/system/wpa_supplicant.service /etc/systemd/system/wpa_supplicant.service
```

* Modification du fichier `/etc/systemd/system/wpa_supplicant.service` :
```
ExecStart=/sbin/wpa_supplicant -u -s -c /etc/wpa_supplicant/wpa_supplicant.conf -i wlo1
Restart=always
```
Commenter la ligne `Alias=dbus-fi.w1.wpa_supplicant1.service` à l'aide du caractère `#`.

* Activer le service `wpa_supplicant` :
```
systemctl enable wpa_supplicant.service
```

* Vérifier le statut du service :
```
iwconfig
```

### 2. Configuration IP

* Modification de la configuration réseau à partir du fichier `/etc/network/interfaces` :
```
auto lo
iface lo inet loopback

auto eno1
allow-hotplug eno1
iface eno1 inet manual

auto wlo1
iface wlo1 inet static
        address 10.0.0.252/24
        gateway 10.0.0.254

auto br0
iface br0 inet static
        bridge_ports eno1
        bridge_waitport 0
        address 10.0.20.254/24
```
&ensp; &ensp; &rarr;  `lo` est l'interface de bouclage  
&ensp; &ensp; &rarr;  `eno1` est l'interface Ethernet  
&ensp; &ensp; &rarr;  `wlo1` est l'interface Wifi  
&ensp; &ensp; &rarr;  `br0` est le pont  
&ensp; &ensp; &rarr;  `10.0.0.0/24` est le réseau des local  
&ensp; &ensp; &rarr;  `10.0.20.0/24` est le réseau des VM

* Ajout des serveurs DNS à partir du fichier `/etc/resolv.conf` :
```
nameserver 10.0.0.254
nameserver 8.8.8.8
```

## Configuration de l'accès réseau pour la VM

* Ajout d'une mascarade pour associer le réseau Wifi à celui de la VM :
```
iptables -t nat -F
iptables -t nat -A POSTROUTING -o wlo1 -s 10.0.20.0/24 -j MASQUERADE
iptables -t nat -L
```
&ensp; &ensp; &rarr;  `wlo1` est l'interface Wifi  
&ensp; &ensp; &rarr;  `10.0.20.0/24` est le réseau des VM

* Sauvegarder la configuration (le paquet `iptables-persistant` doit être installé) :
```
iptables-save > /etc/iptables/rules.v4
```

Si l'ordinateur utilisé a un BIOS UEFI, il faut modifier l'entrée GRUB de Xen pour autoriser son démarrage.  
&ensp; &ensp; &rarr; Modifier le fichier `/etc/grub.d/20_linux_xen` en ajoutant `efi=no-rs` aux arguments de la variable `xen_rm_opts`

## Création d'une machine virtuelle

* Création d’une image pour la VM :
```
xen-create-image --hostname=demineur --ip=10.0.20.1 --gateway=10.0.20.254 --netmask=255.255.255.0 --dir=/usr/local/xen --password=pasglop --dist=buster
```
&ensp; &ensp; &rarr; Dossier de stockage des données de la VM : `/usr/local/xen/domains/demineur`  
&ensp; &ensp; &rarr; Fichier de configuration de la VM : `/etc/xen/demineur.cfg`

* Création des partitions virtuelles :
```
vgcreate storage /dev/sda7
lvcreate -L10G -n demineur-home storage
lvcreate -L10G -n demineur-var storage
```

* Vérification des partitions :
```
lvdisplay
lsblk
```

* Formatage de la partition virtuelle :
```
mkfs.ext4 /dev/storage/demineur-home
mkfs.ext4 /dev/storage/demineur-var
```

* Modification de `/etc/xen/demineur.cfg` :  
&ensp; &ensp; &rarr; Ajout des partitions virtuelles dans la variable `disk` :
```
'phy:/dev/storage/demineur-home,xvda3,w',
'phy:/dev/storage/demineur-var,xvda4,w'
```
&ensp; &ensp; &rarr; Ajout du pont dans dans la variable `vif` :
```
vif         = [ 'mac=00:16:3E:D8:97:68, bridge=br0' ]
```

* Création de la VM :
```
xl create /etc/xen/demineur.cfg
```

* Montage des partitions virtuelles :
```
mount /dev/xvda3 /mnt/xvda3
mount /dev/xvda4 /mnt/xvda4
```

* Copie des données des répertoires `/home` et `/var` :
```
cp -r /home /mnt/xvda3
cp -r /var /mnt/xvda4
```

* Démontage des partitions virtuelles :
```
umount /mnt/xvda3
umount /mnt/xvda4
```

* Ajout des partitions au fichier `/etc/fstab` :
```
/dev/xvda3 /home /ext4 default 0 2
/dev/xvda4 /var /ext4 default 0 2
```

## Utilisation de la VM

* Affichage du mot de passe de la VM :
```
tail -f /var/log/xen-tools/demineur.log
```

* Affichage de l’état des VM :
```
xl list
```

* Connexion à la VM :
```
xen console demineur
```

* Changement du mot de passe :
```
passwd root
```

* Mise à jour de la liste des paquets :
```
apt update
```

## Fichier de configuration complet de la machine virtuelle :

```
#
# Configuration file for the Xen instance demineur, created
# by xen-tools 4.9 on Tue Oct  5 21:07:47 2021.
#

#
#  Kernel + memory size
#


bootloader = 'pygrub'

vcpus       = '1'
memory      = '256'


#
#  Disk device(s).
#
root        = '/dev/xvda2 ro'
disk        = [
                  'file:/usr/local/xen/domains/demineur/disk.img,xvda2,w',
                  'file:/usr/local/xen/domains/demineur/swap.img,xvda1,w',
                  'phy:/dev/storage/demineur-home,xvda3,w',
                  'phy:/dev/storage/demineur-var,xvda4,w'
              ]


#
#  Physical volumes
#


#
#  Hostname
#
name        = 'demineur'

#
#  Networking
#
vif         = [ 'mac=00:16:3E:D8:97:68, bridge=br0' ]

#
#  Behaviour
#
on_poweroff = 'destroy'
on_reboot   = 'restart'
on_crash    = 'restart'
```
