## Serveur SSH

* Modification du fichier `/etc/ssh/sshd_config` :
```
PermitRootLogin yes
```

## Configuration IPv6

* Modification du fichier `/etc/network/interfaces` :
```
iface eth0 inet6 auto
```

## Serveur DNS

* Installation du paquet `bind9`
```
apt install bind9
```

* Modification du fichier `/etc/resolv.conf` :
```
nameserver 127.0.0.1
```

* Modification du fichier `/etc/bind/named.conf.local` :
```
zone "demineur.site" {
	type master;
file "/etc/bind/db.demineur.site";
allow-transfer { 10.0.0.254; };
};
```

* Modification du fichier `/etc/bind/named.conf.options` :
```
options{
  directory "/var/cache/bind";
  forwarders {
     10.0.0.254;
     8.8.8.8;
  };
  listen-on-v6 { any; };
  allow-transfer { "allowed_to_transfer"; };
};
acl "allowed_to_transfer" {
  10.0.0.254/32 ;
};
```

```
cp /etc/bind/db.local /etc/bind/db.demineur.site
```

* Modification du fichier `/etc/bind/db.demineur.site` :
```
;
; BIND data file for demineur.site
;
$TTL    604800
@       IN      SOA     demineur.site. root.demineur.site. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.demineur.site.
@       IN      A       10.0.20.1
@       IN      AAAA    ::1
ns      IN      A       10.0.20.1
www     IN      A       10.0.20.1
```

* Redémarrage du service `bind9` :
```
service bind9 restart
```

### Tester DNS :

* Ajout d'une redirection de ports :
```
iptables -A PREROUTING -t nat -i wlo1 -p udp --dport 53 -j DNAT --to-destination 10.0.20.1:53
```

* Modification du fichier `/etc/resolv.conf`
```
nameserver 10.0.0.252
```

* Vérification de la traduction du nom de domaine `demineur.site` :
```
nslookup demineur.site
```

## Serveur Web

* Installation du paquet `openssl` :
```
apt install openssl
```

* Création d'un certificat TSL :
```
openssl req -nodes -newkey rsa:2048 -sha256 -keyout demineur.site.key -out demineur.site.csr
```

```
mv demineur.site.key /etc/ssl/private
mv demineur.site.csr /etc/ssl/cert
```

Faire signer le certificat demineur.site.csr par une [https://docs.gandi.net/fr/ssl/creation/installation_certif_manuelle.html](https://docs.gandi.net/fr/ssl/creation/installation_certif_manuelle.html) et placer le nouveau certificat (.crt) dans le répertoire /etc/ssl/certs.
Télécharger également le certificat de Gandi GandiStandardSSLCA2.pem et le placer dans le même dossier.

* Installation du paquet `apache2` :
```
apt install apache2
```

* Activation du module SSL :
```
a2enmod ssl
```

* Modification du fichier `/etc/apache2/ports.conf` :
```
<IfModule mod_ssl.c>
   Listen 443
</IfModule>
<IfModule mod_gnutls.c>
   Listen 443
</IfModule>
```

* Modification du fichier `/etc/apache2/sites-available/000-demineur.site-ssl.conf` :
```
<IfModule mod_ssl.c>
        <VirtualHost 10.0.20.1:443>
                ServerName demineur.site
                ServerAlias www.demineur.site
                DocumentRoot /var/www/demineur.site/
                CustomLog /var/log/apache2/secure_access.log combined
                SSLEngine on
                SSLCertificateFile /etc/ssl/certs/demineur.site.crt
SSLCertificateKeyFile /etc/ssl/private/demineur.site.key
SSLCACertificateFile /etc/ssl/certs/GandiStandardSSLCA2.pem
                SSLVerifyClient None
        </VirtualHost>
</IfModule>
```

* Activation du site `demineur.site` :
```
a2ensite 000-demineur.site-ssl
```

* Modification du fichier `nano /etc/apache2/apache2.conf` :
```
ServerName demineur.site
```

* Redémarrage du service `apache2` :
```
service apache2 restart
```

* Modification du fichier `nano /etc/apache2/sites-available/000-default.conf` :
```
Redirect permanent / https://www.demineur.site/
```

* Ajout de redirections de ports :
```
iptables -A PREROUTING -t nat -i wlo1 -p tcp --dport 80 -j DNAT --to-destination 10.0.20.1:80
iptables -A PREROUTING -t nat -i wlo1 -p tcp --dport 443 -j DNAT --to-destination 10.0.20.1:443
```

## Configuration DNSSEC

* Modification du fichier `/etc/bind/named.conf.options` :
```
dnssec-enable yes;
dnssec-validation yes;
dnssec-lookaside auto;
```

* Création du répertoire `demineur.site.dnssec` :
```
mkdir /etc/bind/demineur.site.dnssec/
```

* Génération de la clef asymétrique de signature de clefs de zone :
```
dnssec-keygen -a RSASHA1 -b 2048 -f KSK -n ZONE demineur.site
```

```
mv site-ksk.key demineur.site-ksk.key
mv site-ksk.private demineur.site-ksk.private
```

* Génération de la clef asymétrique de signature des enregistrements :
```
dnssec-keygen -a RSASHA1 -b 1024 -n ZONE demineur.site
```

```
mv site-zsk.key demineur.site-zsk.key
mv site-zsk.private demineur.site-zsk.private
```

* Modification du fichier `/etc/bind/db.demineur.site` :
```
$include /etc/bind/demineur.site.dnssec/demineur.site-ksk.key
$include /etc/bind/demineur.site.dnssec/demineur.site-zsk.key
```

* Signature des enregistrements de la zone :
```
cd /etc/bind/demineur.site.dnssec
dnssec-signzone -o demineur.site -k demineur.site-ksk ../db.demineur.site demineur.site-zsk
```

* Modification du fichier `/etc/bind/named.conf.local` :
```
zone "demineur.site" {
	type master;
file "/etc/bind/db.demineur.site.signed";
allow-transfer { 10.0.0.254; };
};
```

Il ne reste plus qu’à communiquer la partie publique de la KSK (présente dans le fichier demineur.site-ksk.key) à Gandi. L'algorithme utilisé est le 5 (RSA/SHA-1).

### Vérification :

```
dnssec-verify -o demineur.site db.demineur.site.signed
```
