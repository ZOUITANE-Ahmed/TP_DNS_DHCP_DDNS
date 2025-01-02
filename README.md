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

