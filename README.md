# TP_DNS_DHCP_DDNS

Voici les étapes détaillées pour réaliser cette configuration sur Fedora. **Assurez-vous que vous avez les privilèges root** pour exécuter ces tâches.

---

### **Partie 1 : Installation et configuration du serveur DHCP et DNS**

#### **Installation du DHCP**

1. **Installer le serveur DHCP :**
   ```bash
   dnf install dhcp-server -y
   ```

2. **Configurer le fichier `dhcpd.conf` :**
   Ouvrez le fichier `/etc/dhcp/dhcpd.conf` et ajoutez la configuration suivante :
   ```bash
   subnet 192.168.2.0 netmask 255.255.255.0 {
       range 192.168.2.20 192.168.2.119;
       option domain-name "ofppt-maroc.educ";
       option domain-name-servers 192.168.2.200, 192.168.2.254;
       option routers 192.168.2.1;
       option subnet-mask 255.255.255.0;
   }
   ```

3. **Démarrer et activer le service DHCP :**
   ```bash
   systemctl start dhcpd
   systemctl enable dhcpd
   ```

4. **Tester l'allocation IP :**
   Redémarrez une machine cliente pour vérifier qu'elle obtient une IP dans la plage spécifiée.

---

#### **Installation et configuration du serveur DNS (Bind)**

1. **Installer Bind :**
   ```bash
   dnf install bind bind-utils -y
   ```

2. **Configurer le fichier `named.conf` :**
   Ajoutez ou modifiez le fichier `/etc/named.conf` pour inclure les zones directe et inverse :
   ```bash
   zone "ofppt-maroc.educ" IN {
       type master;
       file "/var/named/ma_zone.zone";
       allow-update { none; };
   };

   zone "2.168.192.in-addr.arpa" IN {
       type master;
       file "/var/named/ma_zone.rev";
       allow-update { none; };
   };
   ```

3. **Créer les fichiers de zone :**
   - **Fichier direct : `/var/named/ma_zone.zone`**
     ```bash
     $TTL 86400
     @   IN  SOA srvDNS1.ofppt-maroc.educ. admin.ofppt-maroc.educ. (
             2025010101 ; Serial
             3600       ; Refresh
             1800       ; Retry
             604800     ; Expire
             86400 )    ; Minimum TTL

         IN  NS  srvDNS1.ofppt-maroc.educ.
         IN  NS  srvDNS2.ofppt-maroc.educ.

     srvDNS1 IN  A    192.168.2.200
     srvDNS2 IN  A    192.168.2.254
     PCA     IN  A    192.168.2.10
     PCA     IN  AAAA 2001:abc::10
     www     IN  CNAME PCA
     ```

   - **Fichier inverse : `/var/named/ma_zone.rev`**
     ```bash
     $TTL 86400
     @   IN  SOA srvDNS1.ofppt-maroc.educ. admin.ofppt-maroc.educ. (
             2025010101 ; Serial
             3600       ; Refresh
             1800       ; Retry
             604800     ; Expire
             86400 )    ; Minimum TTL

         IN  NS  srvDNS1.ofppt-maroc.educ.
         IN  NS  srvDNS2.ofppt-maroc.educ.

     200    IN  PTR srvDNS1.ofppt-maroc.educ.
     254    IN  PTR srvDNS2.ofppt-maroc.educ.
     10     IN  PTR PCA.ofppt-maroc.educ.
     ```

4. **Changer les permissions :**
   ```bash
   chown named:named /var/named/ma_zone.*
   ```

5. **Vérifier la configuration :**
   ```bash
   named-checkconf
   named-checkzone ofppt-maroc.educ /var/named/ma_zone.zone
   named-checkzone 2.168.192.in-addr.arpa /var/named/ma_zone.rev
   ```

6. **Démarrer et activer le service DNS :**
   ```bash
   systemctl start named
   systemctl enable named
   ```

7. **Tester avec des commandes :**
   - `nslookup www.ofppt-maroc.educ`
   - `dig www.ofppt-maroc.educ`

---

### **Partie 2 : Configuration d’un serveur DNS secondaire**

1. **Configurer `srvDNS2` en DNS secondaire :**
   Dans `/etc/named.conf` sur `srvDNS2` :
   ```bash
   zone "ofppt-maroc.educ" IN {
       type slave;
       masters { 192.168.2.200; };
       file "/var/named/slave_ofppt.zone";
   };

   zone "2.168.192.in-addr.arpa" IN {
       type slave;
       masters { 192.168.2.200; };
       file "/var/named/slave_ofppt.rev";
   };
   ```

2. **Vérifier le transfert de zone :**
   - Regardez dans `data/named.run` pour confirmer le transfert.

3. **Tester en arrêtant le serveur primaire :**
   ```bash
   systemctl stop named
   ```
   Vérifiez que `srvDNS2` répond aux requêtes DNS avec `dig` ou `nslookup`.

---

### **Partie 3 : Configuration du DDNS avec DHCP**

1. **Autoriser la mise à jour dynamique dans les fichiers de zone :**
   Ajoutez `allow-update { key "rndc-key"; };` aux zones dans `/etc/named.conf`.

2. **Configurer le fichier `/etc/dhcp/dhcpd.conf` pour DDNS :**
   ```bash
   ddns-update-style interim;
   ignore client-updates;
   include "/etc/rndc.key";

   subnet 192.168.2.0 netmask 255.255.255.0 {
       ...
       ddns-domainname "ofppt-maroc.educ";
   }
   ```

3. **Tester les modifications :**
   Redémarrez le service DHCP et vérifiez les mises à jour DNS dans les logs :
   ```bash
   journalctl -u named
   ```

Voici un script shell pour automatiser l'installation et la configuration du DHCP, DNS (primaire et secondaire), et du DDNS sur Fedora. Ce script nécessite des privilèges **root** pour être exécuté.

---

### **Script Shell : `setup_dhcp_dns.sh`**

```bash
#!/bin/bash

# Variables
DOMAIN="ofppt-maroc.educ"
ZONE_DIRECTE="ma_zone.zone"
ZONE_INVERSE="ma_zone.rev"
PRIMARY_DNS="192.168.2.200"
SECONDARY_DNS="192.168.2.254"
DHCP_RANGE_START="192.168.2.20"
DHCP_RANGE_END="192.168.2.119"
GATEWAY="192.168.2.1"
SUBNET="192.168.2.0"
NETMASK="255.255.255.0"

echo ">>> Installation des paquets nécessaires"
dnf install -y dhcp-server bind bind-utils

# Configuration DHCP
echo ">>> Configuration du serveur DHCP"
cat <<EOF > /etc/dhcp/dhcpd.conf
subnet $SUBNET netmask $NETMASK {
    range $DHCP_RANGE_START $DHCP_RANGE_END;
    option domain-name "$DOMAIN";
    option domain-name-servers $PRIMARY_DNS, $SECONDARY_DNS;
    option routers $GATEWAY;
    option subnet-mask $NETMASK;
}
EOF

echo ">>> Démarrage du service DHCP"
systemctl enable --now dhcpd

# Configuration DNS primaire
echo ">>> Configuration du serveur DNS primaire"
cat <<EOF > /etc/named.conf
options {
    directory "/var/named";
    allow-query { any; };
    recursion yes;
};

zone "$DOMAIN" IN {
    type master;
    file "/var/named/$ZONE_DIRECTE";
    allow-update { none; };
};

zone "2.168.192.in-addr.arpa" IN {
    type master;
    file "/var/named/$ZONE_INVERSE";
    allow-update { none; };
};
EOF

# Création des fichiers de zone
echo ">>> Création des fichiers de zone DNS"
cat <<EOF > /var/named/$ZONE_DIRECTE
\$TTL 86400
@   IN  SOA srvDNS1.$DOMAIN. admin.$DOMAIN. (
        2025010201 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL
    IN  NS  srvDNS1.$DOMAIN.
    IN  NS  srvDNS2.$DOMAIN.
srvDNS1 IN  A    $PRIMARY_DNS
srvDNS2 IN  A    $SECONDARY_DNS
PCA     IN  A    192.168.2.10
PCA     IN  AAAA 2001:abc::10
www     IN  CNAME PCA
EOF

cat <<EOF > /var/named/$ZONE_INVERSE
\$TTL 86400
@   IN  SOA srvDNS1.$DOMAIN. admin.$DOMAIN. (
        2025010201 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL
    IN  NS  srvDNS1.$DOMAIN.
    IN  NS  srvDNS2.$DOMAIN.
200    IN  PTR srvDNS1.$DOMAIN.
254    IN  PTR srvDNS2.$DOMAIN.
10     IN  PTR PCA.$DOMAIN.
EOF

echo ">>> Changement des permissions des fichiers de zone"
chown named:named /var/named/$ZONE_DIRECTE /var/named/$ZONE_INVERSE

echo ">>> Vérification des fichiers de configuration DNS"
named-checkconf
named-checkzone $DOMAIN /var/named/$ZONE_DIRECTE
named-checkzone 2.168.192.in-addr.arpa /var/named/$ZONE_INVERSE

echo ">>> Démarrage du service DNS"
systemctl enable --now named

# Configuration DNS secondaire (à exécuter sur le serveur secondaire)
if [ "$1" == "secondary" ]; then
    echo ">>> Configuration du serveur DNS secondaire"
    cat <<EOF > /etc/named.conf
options {
    directory "/var/named";
    allow-query { any; };
    recursion yes;
};

zone "$DOMAIN" IN {
    type slave;
    masters { $PRIMARY_DNS; };
    file "/var/named/slave_$ZONE_DIRECTE";
};

zone "2.168.192.in-addr.arpa" IN {
    type slave;
    masters { $PRIMARY_DNS; };
    file "/var/named/slave_$ZONE_INVERSE";
};
EOF
    echo ">>> Démarrage du service DNS secondaire"
    systemctl enable --now named
    echo ">>> Transfert de zone vérifié dans /var/named/slave_*"
fi

# Configuration DDNS
echo ">>> Configuration DDNS"
cat <<EOF >> /etc/named.conf
key "rndc-key" {
    algorithm hmac-md5;
    secret "random-secret-key";
};

zone "$DOMAIN" IN {
    type master;
    file "/var/named/$ZONE_DIRECTE";
    allow-update { key "rndc-key"; };
};

zone "2.168.192.in-addr.arpa" IN {
    type master;
    file "/var/named/$ZONE_INVERSE";
    allow-update { key "rndc-key"; };
};
EOF

cat <<EOF >> /etc/dhcp/dhcpd.conf
ddns-update-style interim;
ignore client-updates;
include "/etc/rndc.key";
EOF

echo ">>> Redémarrage des services DHCP et DNS"
systemctl restart dhcpd named

echo ">>> Configuration terminée ! Testez avec les commandes nslookup, dig et ping."
```

---

### **Utilisation**

1. **Exécuter sur le serveur primaire :**
   ```bash
   bash setup_dhcp_dns.sh
   ```

2. **Exécuter sur le serveur secondaire :**
   ```bash
   bash setup_dhcp_dns.sh secondary
   ```

---

### **Tests**

- Pour tester le DNS primaire :
  ```bash
  nslookup www.ofppt-maroc.educ
  dig www.ofppt-maroc.educ
  ```

- Pour tester le transfert de zone sur le DNS secondaire :
  Vérifiez que les fichiers `slave_ma_zone.zone` et `slave_ma_zone.rev` sont créés dans `/var/named`.

---


