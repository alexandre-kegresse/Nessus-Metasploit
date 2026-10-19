# Job 01 — Présentation de Nessus

## C'est quoi

Nessus (édité par **Tenable**, anciennement « Nessus Professional », aujourd'hui aussi
proposé en version cloud « Tenable Vulnerability Management ») est un **scanner de
vulnérabilités**. On lui donne une ou plusieurs cibles (adresses IP), il interroge les
services qui tournent dessus et il dit lesquels présentent des failles connues, des
mauvaises configurations ou des correctifs manquants.

Pour mon lab j'utilise **Nessus Essentials**, la version gratuite (jusqu'à 16 adresses IP),
largement suffisante pour scanner une machine.

## Comment ça marche

1. **Découverte** : Nessus repère la machine (ping, scan de ports) et identifie les
   services qui écoutent (FTP, SSH, SMB, HTTP…) et leurs versions.
2. **Plugins** : pour chaque service/version, Nessus lance des **plugins**. Un plugin est
   un petit test associé à une faille connue (souvent une référence **CVE**). C'est la base
   de connaissances de Nessus, mise à jour très régulièrement.
3. **Résultats** : chaque problème trouvé devient un **finding**, classé par sévérité.

## Lire les résultats : la sévérité

Nessus classe les findings selon le score **CVSS** :

| Sévérité | Couleur | Idée |
|---|---|---|
| Critical | rouge foncé | à corriger en priorité absolue |
| High | rouge | grave |
| Medium | orange | à traiter |
| Low | vert | mineur |
| Info | bleu | information, pas une faille |

Pour chaque finding, Nessus donne : la description, la **CVE**, le score CVSS, la preuve
(ce qu'il a observé) et surtout une **solution** (mettre à jour, désactiver le service…).

## Types de scans utiles

- **Host Discovery** : juste repérer les machines vivantes.
- **Basic Network Scan** : le scan généraliste, celui que j'utilise pour le Job 04.
- **Advanced Scan** : pour régler finement les options.
- Scan **avec authentification** (credentialed) : on donne un compte à Nessus, il voit
  bien plus (correctifs manquants réels) — plus précis et moins « bruyant » qu'un scan
  externe.

## Ses limites (à dire en soutenance)

- Nessus **détecte**, il n'exploite pas : il dit « cette faille existe probablement »,
  c'est Metasploit qui confirme en l'exploitant réellement.
- Des **faux positifs** sont possibles (d'où l'intérêt du scan authentifié et de la
  vérification manuelle).
- Un scan génère du trafic et peut perturber un service fragile : on scanne dans un cadre
  autorisé et maîtrisé.

## En une phrase

Nessus est l'outil d'**audit / détection** (la photo de l'état de sécurité d'une machine) ;
Metasploit est l'outil d'**exploitation** (la preuve qu'une faille est réellement
exploitable). Les deux se complètent : on détecte, puis on prouve, puis on corrige.
