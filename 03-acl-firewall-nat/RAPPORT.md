# Networking Essentials – Projet 3 : Routeur / Pare-feu (NAT + ACL)

**Classe :** E1A – Geneva Institute of Technology
**Auteur :** Rrudhani Deaa
**Date :** 13 octobre 2026
**Outil :** GNS3 (routeur Cisco c7200, Ethernet switch, VPCS, nœud NAT)

---

[TOC]

---

## 1. Objectif du projet

Configurer le routeur **R1** pour qu'il :

1. relie le réseau local de l'entreprise (LAN) à Internet (WAN) ;
2. traduise les adresses privées du LAN en une adresse publique (**NAT overload / PAT**) ;
3. joue le rôle de **pare-feu** grâce à une ACL étendue qui :
   - bloque le **HTTP non chiffré (port 80)** ;
   - bloque le **Telnet (port 23)** ;
   - autorise tout le reste (HTTPS 443, DNS, ping…).

On vérifie le fonctionnement **avant** et **après** la mise en place du pare-feu.

---

## 2. Topologie et plan d'adressage

![Capture d'écran 2026-10-13 180244](https://hackmd.io/_uploads/Sk2fhAojMe.png)


| Équipement | Interface | Relié à | Adresse IP | Passerelle | Rôle |
|---|---|---|---|---|---|
| NAT1 | nat0 | R1 g1/0 | 192.168.209.2 | – | Accès Internet + serveur DHCP |
| R1 | Gi1/0 | NAT1 nat0 | 192.168.209.157/24 (DHCP) | 192.168.209.2 | WAN – Internet (NAT outside) |
| R1 | Gi0/0 | Switch1 e3 | 192.168.10.1/24 | – | LAN – Réseau entreprise (NAT inside) |
| Switch1 | e1, e2, e3 | PC1, PC2, R1 | – | – | Relie les équipements du LAN |
| PC1 | e0 | Switch1 e1 | 192.168.10.11/24 | 192.168.10.1 | Poste utilisateur |
| PC2 | e0 | Switch1 e2 | 192.168.10.12/24 | 192.168.10.1 | Poste utilisateur |

- **Réseau LAN :** 192.168.10.0/24
- **Réseau WAN :** 192.168.209.0/24 (adresse donnée automatiquement par DHCP)
- **DNS des PC :** 8.8.8.8 (Google)

---

## 3. Configuration de R1

### 3.1 Configuration des interfaces

```
R1#enable
R1#configure terminal
R1(config)#hostname R1
R1(config)#interface GigabitEthernet0/0
R1(config-if)# description LAN - Reseau entreprise
R1(config-if)# ip address 192.168.10.1 255.255.255.0
R1(config-if)# ip nat inside
R1(config-if)# no shutdown
R1(config-if)#exit
R1(config)#interface GigabitEthernet1/0
R1(config-if)# ip address dhcp
R1(config-if)# ip nat outside
R1(config-if)# no shutdown
R1(config-if)#exit
R1(config)#end
R1#write memory
```




| Commande | Explication |
|---|---|
| `enable` | Passe en mode privilégié (`R1#`), nécessaire pour configurer et sauvegarder. |
| `configure terminal` | Entre en mode de configuration globale (`R1(config)#`). |
| `hostname R1` | Donne le nom **R1** au routeur (apparaît dans l'invite de commande). |
| `interface GigabitEthernet0/0` | Sélectionne l'interface côté LAN (`R1(config-if)#`). |
| `description LAN - Reseau entreprise` | Ajoute un commentaire sur l'interface pour savoir à quoi elle sert. Visible avec `show interfaces description`. |
| `ip address 192.168.10.1 255.255.255.0` | Attribue l'IP de la passerelle du LAN. C'est l'adresse que les PC utilisent comme *gateway*. |
| `ip nat inside` | Indique que cette interface est **à l'intérieur** du NAT : les adresses privées qui arrivent ici seront traduites. |
| `no shutdown` | Active l'interface (sur Cisco, elles sont éteintes par défaut). Le message `%LINK-3-UPDOWN ... changed state to up` confirme l'activation. |
| `interface GigabitEthernet1/0` | Sélectionne l'interface côté WAN (vers Internet). |
| `ip address dhcp` | L'interface demande automatiquement une IP au serveur DHCP de NAT1. Le message `%DHCP-6-ADDRESS_ASSIGN ... 192.168.209.157` confirme l'adresse reçue. |
| `ip nat outside` | Indique que cette interface est **à l'extérieur** du NAT : c'est son adresse qui sera utilisée vers Internet. |
| `exit` / `end` | `exit` remonte d'un niveau, `end` revient directement en mode privilégié. |
| `write memory` | Sauvegarde la configuration (running-config → startup-config) pour qu'elle survive à un redémarrage. |

![Capture d'écran 2026-10-13 170941](https://hackmd.io/_uploads/B1ZZvZCifx.png)




**Message `Warning: Attempting to overwrite an NVRAM configuration previously written by a different version of the system image`**
Ce message apparaît car la NVRAM contenait une sauvegarde faite avec une autre version d'IOS. Il suffit de confirmer avec **Entrée** (`[confirm]`) : ce n'est pas une erreur.


**Vérification :**

```
R1#show ip interface brief
R1#show interfaces description
```

| Commande | Explication |
|---|---|
| `show ip interface brief` | Résumé des interfaces : IP, méthode (`manual` = tapée à la main, `DHCP` = reçue automatiquement), état `up/up`. |
| `show interfaces description` | Affiche la description de chaque interface. |

Résultat : Gi0/0 = 192.168.10.1 (manual, up/up), Gi1/0 = 192.168.209.157 (DHCP, up/up). La description de Gi1/0 est **vide** : elle avait été oubliée (corrigé en 3.2).

![Capture d'écran 2026-10-13 171038](https://hackmd.io/_uploads/HJhonAiozl.png)


### 3.2 Ajout de la description WAN

Première tentative : la commande `interface GigabitEthernet1/0` a été tapée directement en mode privilégié (`R1#`), ce qui donne une erreur.

![Capture d'écran 2026-10-13 172156](https://hackmd.io/_uploads/r1ZC3Aijzg.png)

**Cause :** la commande `interface` n'existe qu'en mode configuration. Il faut d'abord taper `configure terminal`.

```
R1#configure terminal
R1(config)#interface GigabitEthernet1/0
R1(config-if)#description WAN - Internet
R1(config-if)#end
R1#show interfaces description
```

| Commande | Explication |
|---|---|
| `configure terminal` | Passe en mode configuration (l'étape oubliée). |
| `description WAN - Internet` | Indique que Gi1/0 est l'interface de sortie vers Internet. |
| `show interfaces description` | Vérifie que la description apparaît bien. |


![Capture d'écran 2026-10-13 172311](https://hackmd.io/_uploads/SyLka0sozx.png)

### 3.3 Route par défaut

```
R1#show ip route
```

```
Gateway of last resort is 192.168.209.2 to network 0.0.0.0
S*    0.0.0.0/0 [254/0] via 192.168.209.2
C        192.168.10.0/24 is directly connected, GigabitEthernet0/0
L        192.168.10.1/32 is directly connected, GigabitEthernet0/0
C        192.168.209.0/24 is directly connected, GigabitEthernet1/0
L        192.168.209.157/32 is directly connected, GigabitEthernet1/0
```

Aucune route n'a été tapée à la main : avec `ip address dhcp`, le routeur reçoit aussi la **passerelle** (192.168.209.2) et crée **automatiquement** une route par défaut.

| Élément | Signification |
|---|---|
| `S*` | Route statique (`S`) qui est la route par défaut (`*` = *candidate default*). |
| `0.0.0.0/0` | « Toutes les destinations » : tout ce qui n'est pas connu part par là. |
| `[254/0]` | Distance administrative **254** = route apprise par DHCP (très peu prioritaire), métrique 0. |
| `via 192.168.209.2` | Prochain saut : la passerelle de NAT1. |
| `C` | Réseau directement connecté à une interface. |
| `L` | Adresse propre de l'interface (en /32). |


![Capture d'écran 2026-10-13 171047](https://hackmd.io/_uploads/Bk9xLZRsMl.png)
![Capture d'écran 2026-10-13 171038](https://hackmd.io/_uploads/Sk5gI-Cife.png)



### 3.4 Configuration du NAT (PAT / overload)

```
R1#configure terminal
R1(config)#access-list 1 permit 192.168.10.1 0.0.0.255
R1(config)#ip nat inside source list 1 interface GigabitEthernet1/0 overload
R1(config)#end
R1#write memory
R1#ping 8.8.8.8
```
![Capture d'écran 2026-10-13 171227](https://hackmd.io/_uploads/rJWTDZRiMx.png)


| Commande | Explication |
|---|---|
| `access-list 1 permit 192.168.10.1 0.0.0.255` | ACL **standard** n°1 qui dit **quelles adresses ont le droit d'être traduites** par le NAT. Le wildcard `0.0.0.255` signifie « les 3 premiers octets doivent correspondre, le dernier peut être n'importe quoi » → tout le réseau 192.168.10.x. |
| `ip nat inside source list 1 interface GigabitEthernet1/0 overload` | Active le NAT : les adresses **source** venant de l'intérieur (`inside source`) et autorisées par l'**ACL 1** sont remplacées par l'adresse de **Gi1/0**. `overload` = PAT : plusieurs PC partagent une seule IP publique, différenciés par le **numéro de port**. |
| `write memory` | Sauvegarde la configuration. |
| `ping 8.8.8.8` | Test depuis le routeur : `!!!!!` = 5 réponses sur 5 (100 %), Internet est joignable par R1. |



:::info
**Remarque :** l'ACL a été tapée avec `192.168.10.1`, mais `show access-lists` affiche `192.168.10.0`. Avec le wildcard 0.0.0.255, IOS ignore le dernier octet et corrige automatiquement l'adresse en **adresse réseau**. Le résultat est donc correct.
:::

### 3.5 Création et application du pare-feu (ACL étendue nommée)

**Avant le pare-feu**, seule l'ACL 1 (celle du NAT) existe :

```
R1#write memory
R1#show access-lists
Standard IP access list 1
    10 permit 192.168.10.0, wildcard bits 0.0.0.255 (7 matches)
```
![Capture d'écran 2026-10-13 172352](https://hackmd.io/_uploads/HyakYW0ozl.png)




**Création du pare-feu :**

```
R1#configure terminal
R1(config)#ip access-list extended PAREFEU-LAN
R1(config-ext-nacl)#remark Bloquer HTTP non chiffre
R1(config-ext-nacl)#deny tcp 192.168.10.0 0.0.0.255 any eq 80
R1(config-ext-nacl)#remark Bloquer Telnet
R1(config-ext-nacl)#deny tcp 192.168.10.0 0.0.0.255 any eq 23
R1(config-ext-nacl)#remark Autoriser tout le reste
R1(config-ext-nacl)#permit ip 192.168.10.0 0.0.0.255 any
R1(config-ext-nacl)#exit
R1(config)#interface GigabitEthernet0/0
R1(config-if)# ip access-group PAREFEU-LAN in
R1(config-if)#end
R1#write memory
R1#show access-lists
```
![Capture d'écran 2026-10-13 172526](https://hackmd.io/_uploads/S1rMYbCoGe.png)

| Commande | Explication |
|---|---|
| `ip access-list extended PAREFEU-LAN` | Crée une ACL **étendue** et **nommée** PAREFEU-LAN. Une ACL étendue peut filtrer selon la source, la destination, le protocole **et le port**, contrairement à une ACL standard (source seulement). |
| `remark ...` | Ajoute un commentaire dans l'ACL pour expliquer chaque règle. N'a aucun effet sur le trafic. |
| `deny tcp 192.168.10.0 0.0.0.255 any eq 80` | **Refuse** le trafic **TCP** venant du LAN vers **n'importe quelle destination** (`any`) sur le **port 80** (HTTP non chiffré). IOS l'affiche ensuite sous le nom `www`. |
| `deny tcp 192.168.10.0 0.0.0.255 any eq 23` | Refuse le **Telnet** (port 23), protocole d'administration à distance non chiffré (mots de passe en clair). IOS l'affiche sous le nom `telnet`. |
| `permit ip 192.168.10.0 0.0.0.255 any` | **Autorise** tout le reste du trafic IP du LAN (HTTPS, DNS, ICMP…). Sans cette ligne, tout serait bloqué à cause du *deny implicite*. |
| `interface GigabitEthernet0/0` | Sélectionne l'interface LAN. |
| `ip access-group PAREFEU-LAN in` | Applique l'ACL sur Gi0/0 dans le sens **entrant** (`in`) : les paquets sont filtrés dès qu'ils arrivent des PC, avant d'être routés. |
| `show access-lists` | Vérifie que l'ACL PAREFEU-LAN existe avec ses 3 règles (10, 20, 30), encore à 0 match. |



**Points importants sur les ACL**
- Les règles sont lues **de haut en bas** et la **première qui correspond** est appliquée. Les `deny` doivent donc être placés **avant** le `permit`.
- Toute ACL se termine par un **deny any implicite** (invisible) : ce qui n'est pas autorisé est bloqué.
- Une ACL étendue se place **au plus près de la source** (ici l'interface LAN, en entrée) pour éviter de faire circuler du trafic inutile dans le routeur.
- IOS numérote les règles **10, 20, 30**, ce qui permet d'en insérer plus tard entre deux (ex. 15).
:::

---

## 4. Configuration des PC (VPCS)

| Commande | Explication |
|---|---|
| `ip 192.168.10.11/24 192.168.10.1` | Donne au PC son adresse IP, son masque (/24 = 255.255.255.0) et sa **passerelle** (R1). `Checking for duplicate address...` vérifie qu'aucun autre appareil n'utilise déjà cette IP. |
| `ip dns 8.8.8.8` | Définit le serveur DNS (Google) pour pouvoir utiliser des noms comme `google.com`. |
| `save` | Sauvegarde la configuration du PC dans `startup.vpc`. |
| `show ip` | Affiche la configuration : nom, IP/masque, passerelle, DNS, adresse MAC. |

### 4.1 PC1

```
PC1> ip 192.168.10.11/24 192.168.10.1
PC1> ip dns 8.8.8.8
PC1> save
PC1> show ip
```

![Capture d'écran 2026-10-13 171314](https://hackmd.io/_uploads/Hkux5WAiMe.png)

### 4.2 PC2

```
PC2> ip 192.168.10.12/24 192.168.10.1
PC2> ip dns 8.8.8.8
PC2> save
PC2> show ip
```

![Capture d'écran 2026-10-13 171350](https://hackmd.io/_uploads/Hylf5ZAofx.png)



---

## 5. Tests avant le pare-feu

### 5.1 Tests depuis PC1

```
PC1> ping 192.168.10.1
PC1> ping 8.8.8.8
PC1> ping google.com
PC1> ping google.com -P 6 -p 443
PC1> ping google.com -P 6 -p 80
```
![Capture d'écran 2026-10-13 171442](https://hackmd.io/_uploads/S12C9ZCozg.png)

| Commande | Explication |
|---|---|
| `ping 192.168.10.1` | Teste la liaison PC1 → passerelle (R1). |
| `ping 8.8.8.8` | Teste l'accès à Internet par IP (vérifie routage + NAT). |
| `ping google.com` | Teste le **DNS** : le nom est résolu en `172.217.22.110` puis pingé. |
| `ping google.com -P 6 -p 443` | `-P 6` = utiliser **TCP** (protocole n°6) au lieu d'ICMP, `-p 443` = port de destination **443 (HTTPS)**. Simule une connexion à un site web sécurisé. |
| `ping google.com -P 6 -p 80` | Même chose sur le port **80 (HTTP)**. |

| Test | Résultat | Conclusion |
|---|---|---|
| Ping passerelle | Réponse (ttl=255) | LAN OK |
| Ping 8.8.8.8 | Réponse (ttl=127) | Routage + NAT OK |
| Ping google.com | Résolu en 172.217.22.110 + réponse | DNS OK |
| TCP 443 | `Connect` / `SendData` / `Close` | HTTPS ouvert |
| TCP 80 | `Connect` / `SendData` / `Close` (3 fois) | HTTP ouvert |

- **Connect / SendData / Close** : la connexion TCP s'établit (poignée de main), des données sont envoyées, puis la connexion est fermée proprement → le port est **ouvert**.
- La dernière ligne `Connect ... timeout` est normale avec VPCS : le serveur ne répond plus aux tentatives suivantes. L'important est que la première séquence soit complète.
- **ttl=255** = réponse du routeur lui-même ; **ttl=127** = réponse qui vient d'Internet (elle a traversé des routeurs).



### 5.2 Table NAT

```
R1#show ip nat translations
```

![Capture d'écran 2026-10-13 171506](https://hackmd.io/_uploads/S1fVoZAsfg.png)

| Colonne | Signification | Exemple |
|---|---|---|
| **Pro** | Protocole | tcp, udp, icmp |
| **Inside global** | Adresse **publique** après traduction (Gi1/0) + port attribué | 192.168.209.157:4096 |
| **Inside local** | Adresse **privée** du PC | 192.168.10.11:14503 |
| **Outside local / global** | Adresse du serveur joint | 172.217.22.110:443 |

On observe :
- **udp → 8.8.8.8:53** : requêtes **DNS** (résolution de google.com) ;
- **icmp** : les **ping** vers 8.8.8.8 et google.com ;
- **tcp → :443** et **tcp → :80** : connexions HTTPS et HTTP.

Toutes les connexions de PC1 sortent avec **la même adresse 192.168.209.157** mais des **ports différents** (4096, 4097, 4501…) : c'est le principe du **PAT (overload)**.



---

## 6. Tests après le pare-feu

### 6.1 Tests depuis PC1

```
PC1> ping 8.8.8.8
PC1> ping google.com
PC1> ping google.com -P 6 -p 443
PC1> ping google.com -P 6 -p 80
```

| Test | Résultat | Conclusion |
|---|---|---|
| Ping 8.8.8.8 | Réponse | ICMP autorisé (règle 30) |
| Ping google.com | Résolu + réponse | DNS autorisé (règle 30) |
| TCP 443 | `Connect` / `SendData` / `Close` | HTTPS **toujours autorisé** |
| TCP 80 | `*192.168.10.1 ... (ICMP type:3, code:13, Communication administratively prohibited)` | HTTP **bloqué** par la règle 10 |

![Capture d'écran 2026-10-13 172645](https://hackmd.io/_uploads/r1pIsbAoMe.png)



**ICMP type 3, code 13 – Communication administratively prohibited**
C'est R1 (192.168.10.1) qui répond à PC1 : « ta demande est **interdite par l'administrateur** ». Le type 3 signifie *destination injoignable* et le code 13 précise que la cause est un **filtre (ACL)**. C'est la preuve que le blocage vient bien du pare-feu et non d'une panne.


**Vérification des compteurs sur R1 :**

```
R1#show access-lists
```

Avant les tests : aucun match sur PAREFEU-LAN. Après les tests : **3 matches** sur la règle 10 (www) et **14** sur la règle 30 (permit).

![Capture d'écran 2026-10-13 172731](https://hackmd.io/_uploads/S1qqj-Rszg.png)


### 6.2 Test bonus : Telnet (port 23) depuis PC1

```
PC1> ping google.com -P 6 -p 23
```
![Capture d'écran 2026-10-13 172803](https://hackmd.io/_uploads/HyMhjb0jGx.png)

| Commande | Explication |
|---|---|
| `ping google.com -P 6 -p 23` | Tente une connexion TCP sur le port **23 (Telnet)**. |

Résultat : 3 réponses `ICMP type:3, code:13, Communication administratively prohibited` venant de 192.168.10.1 → le **Telnet est bloqué** par la règle 20.



Vérification sur R1 : la règle 20 (telnet) passe de **0 à 3 matches**.

![Capture d'écran 2026-10-13 172825](https://hackmd.io/_uploads/B1p6s-RoGe.png)


### 6.3 Test depuis PC2 (port 80)

L'ACL vise tout le réseau 192.168.10.0/24 (wildcard 0.0.0.255), donc PC2 doit être bloqué de la même façon.

```
PC2> ping 8.8.8.8 -P 6 -p 80
```
![Capture d'écran 2026-10-13 173103](https://hackmd.io/_uploads/S1klhWAsGl.png)

Résultat : `ICMP type:3, code:13, Communication administratively prohibited` puis `Connect 80@8.8.8.8 timeout` → le HTTP est aussi **bloqué pour PC2**.


Vérification sur R1 : la règle 10 (www) passe de **3 à 5 matches** → le pare-feu s'applique à **tout le LAN**.

![Capture d'écran 2026-10-13 173116](https://hackmd.io/_uploads/SkGz2WAoze.png)


---

## 7. Analyse des compteurs de l'ACL

| Moment | Capture | ACL 1 (NAT) | Règle 10 – www | Règle 20 – telnet | Règle 30 – permit |
|---|---|---|---|---|---|
| Juste après création | 10 / 16 | 7 | 0 | 0 | 0 |
| Après tests PC1 | 16 | 14 | **3** | 0 | 14 |
| Après test Telnet PC1 | 18 | 15 | 3 | **3** | 16 |
| Après test port 80 PC2 | 20 | 15 | **5** | 3 | 16 |

- **Règle 10 (www)** : 3 matches = les tentatives HTTP de PC1 bloquées, puis +2 avec le test de PC2 → le blocage concerne **les deux PC**.
- **Règle 20 (telnet)** : passe de 0 à 3 après le test bonus → le Telnet est bien bloqué. Le PC renvoie sa demande de connexion plusieurs fois avant d'abandonner, d'où 3 matches.
- **Règle 30 (permit)** : augmente avec le trafic autorisé (ping, DNS, HTTPS). Elle ne bouge pas pendant le test de PC2 → rien d'autre n'est passé.
- **ACL 1** : compte les paquets traduits par le NAT ; elle augmente avec le trafic sortant autorisé.
- Le nombre de matches peut varier (3 pour PC1, 2 pour PC2) selon le nombre de tentatives TCP envoyées par VPCS. Ce qui compte, c'est que **le compteur augmente sur la bonne règle**.

---

## 8. Problèmes rencontrés

| Problème | Cause | Solution | Capture |
|---|---|---|---|
| `% Invalid input detected at '^' marker` en tapant `interface GigabitEthernet1/0` | Commande tapée en mode **privilégié** (`R1#`) au lieu du mode **configuration** (`R1(config)#`). | Taper `configure terminal` avant `interface ...`. | 4 |
| Gi1/0 sans description | Description oubliée lors de la configuration initiale. | Ajout de `description WAN - Internet`. | 3, 5 |
| Avertissement NVRAM lors du `write memory` | Ancienne sauvegarde faite avec une autre version d'IOS. | Confirmer avec Entrée, la sauvegarde se fait normalement (`[OK]`). | 2 |

---

## 9. Conclusion

Le routeur R1 assure maintenant trois fonctions :

1. **Routage** entre le LAN 192.168.10.0/24 et Internet, grâce à la route par défaut reçue par DHCP ;
2. **NAT overload (PAT)** : les PC du LAN sortent tous avec l'adresse 192.168.209.157, différenciés par les ports ;
3. **Pare-feu** avec l'ACL étendue PAREFEU-LAN :
   - HTTP (80) et Telnet (23) **bloqués**, avec un message ICMP *administratively prohibited* ;
   - HTTPS, DNS et ping **autorisés**.

Les tests avant/après et les compteurs `show access-lists` prouvent que chaque règle fonctionne, pour PC1 comme pour PC2.

---

## 10. Annexe – Commandes par équipement

### 10.1 NAT1

Aucune commande. Nœud intégré à GNS3 qui donne l'accès à Internet et fournit une IP par **DHCP** à Gi1/0 de R1.

### 10.2 R1

```
enable
configure terminal
hostname R1
interface GigabitEthernet0/0
 description LAN - Reseau entreprise
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
 no shutdown
 exit
interface GigabitEthernet1/0
 description WAN - Internet
 ip address dhcp
 ip nat outside
 no shutdown
 exit
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet1/0 overload
ip access-list extended PAREFEU-LAN
 remark Bloquer HTTP non chiffre
 deny tcp 192.168.10.0 0.0.0.255 any eq 80
 remark Bloquer Telnet
 deny tcp 192.168.10.0 0.0.0.255 any eq 23
 remark Autoriser tout le reste
 permit ip 192.168.10.0 0.0.0.255 any
 exit
interface GigabitEthernet0/0
 ip access-group PAREFEU-LAN in
 end
write memory
```

Vérification :

```
show ip interface brief
show interfaces description
show ip route
ping 8.8.8.8
show ip nat translations
show access-lists
```

### 10.3 Switch1

Aucune commande. L'Ethernet switch de GNS3 n'a pas de ligne de commande : tous les ports sont en **access VLAN 1** par défaut.

### 10.4 PC1

```
ip 192.168.10.11/24 192.168.10.1
ip dns 8.8.8.8
save
show ip
```

Tests avant pare-feu :

```
ping 192.168.10.1
ping 8.8.8.8
ping google.com
ping google.com -P 6 -p 443
ping google.com -P 6 -p 80
```

Tests après pare-feu :

```
ping 8.8.8.8
ping google.com
ping google.com -P 6 -p 443
ping google.com -P 6 -p 80
ping google.com -P 6 -p 23
```

### 10.5 PC2

```
ip 192.168.10.12/24 192.168.10.1
ip dns 8.8.8.8
save
show ip
```

Test après pare-feu :

```
ping 8.8.8.8 -P 6 -p 80
```

---

## 11. Liste des captures

| N° | Section | Contenu | Comment la reconnaître |
|---|---|---|---|
| 1 | 2 | Topologie GNS3 | Le schéma avec NAT1, R1, Switch1, PC1, PC2 |
| 2 | 3.1 | Configuration des interfaces de R1 | Commence par `R1#enable`, finit par `[OK]` |
| 3 | 3.1 | `show ip interface brief` + `show interfaces description` | Gi1/0 sans description |
| 4 | 3.2 | Erreur `Invalid input detected` | 4 lignes `% Invalid input` |
| 5 | 3.2 | Ajout description WAN | `description WAN - Internet` + tableau avec LAN et WAN |
| 6 | 3.3 | `show ip route` | Commence directement par `show ip route` |
| 7 | 3.3 | `show interfaces description` + `show ip route` | Description en haut, route en dessous |
| 8 | 3.4 | Config NAT + ping 8.8.8.8 depuis R1 | `access-list 1 permit ...` + `Success rate is 100 percent` |
| 9 | 3.5 | `show access-lists` avant pare-feu | Seulement `Standard IP access list 1 (7 matches)` |
| 10 | 3.5 | Création + application de PAREFEU-LAN | `ip access-list extended PAREFEU-LAN` |
| 11 | 4.1 | Configuration PC1 | `PC1> ip 192.168.10.11/24 ...` |
| 12 | 4.2 | Configuration PC2 | `PC2> ip 192.168.10.12/24 ...` |
| 13 | 5.1 | Tests PC1 avant pare-feu | `PC1> ping 192.168.10.1`, port 80 ouvert |
| 14 | 5.2 | `show ip nat translations` | Tableau Inside global / Inside local |
| 15 | 6.1 | Tests PC1 après pare-feu | Port 80 `administratively prohibited` |
| 16 | 6.1 | `show access-lists` avant/après tests PC1 | 7 matches puis 14 matches, `www (3 matches)` |
| 17 | 6.2 | Test Telnet PC1 | `ping google.com -P 6 -p 23` |
| 18 | 6.2 | `show access-lists` après Telnet | `telnet (3 matches)` |
| 19 | 6.3 | Test port 80 PC2 | `PC2> ping 8.8.8.8 -P 6 -p 80` |
| 20 | 6.3 | `show access-lists` final | `www (5 matches)` |