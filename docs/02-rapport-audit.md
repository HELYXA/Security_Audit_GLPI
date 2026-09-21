# Rapport d'audit de sécurité — Instance GLPI (gabarit)

> Ce document est un **gabarit de rapport d'audit**, structuré à partir d'un pentest mené en labo personnel sur une instance GLPI. Les résultats de scan originaux (sorties nmap/nikto/sqlmap/OpenVAS) ne sont plus disponibles ; ce fichier documente donc la **structure de rapport** et les **vecteurs de test couverts**, réutilisables pour un nouvel audit.

## 1. Introduction

**Objectif de l'audit** : évaluer la posture de sécurité d'une instance GLPI (application, configuration, serveur hôte) déployée en environnement de laboratoire, selon une approche de type pentest boîte grise (accès réseau à la cible, sans compte applicatif fourni au départ).

**Méthodologie utilisée** : approche en 6 phases — reconnaissance, évaluation de configuration, analyse de vulnérabilités connues, tests de permissions/accès, exploitation et élévation de privilèges, sécurité du serveur hôte. Détail complet dans [`01-methodologie.md`](01-methodologie.md).

**Périmètre** : instance GLPI `glpi-lab.local` (`192.168.56.101`) et son serveur hôte, en environnement de laboratoire isolé (VirtualBox). Hors périmètre : tout système tiers du réseau.

## 2. Barème de sévérité (CVSS)

Chaque vulnérabilité confirmée est destinée à être notée selon le score **CVSS v3.1** (Common Vulnerability Scoring System), qui va de 0 à 10 :

| Score CVSS | Sévérité |
|---|---|
| 9.0 – 10.0 | Critique |
| 7.0 – 8.9 | Élevée |
| 4.0 – 6.9 | Moyenne |
| 0.1 – 3.9 | Faible |
| 0.0 | Aucune |

Calculateur officiel : [nvd.nist.gov/vuln-metrics/cvss/v3-calculator](https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator)

## 3. Format d'une fiche de vulnérabilité

Chaque vulnérabilité confirmée lors d'un audit doit être documentée selon ce format :

| Champ | Description |
|---|---|
| **Composant / URL** | Endpoint ou fichier concerné |
| **Catégorie CWE** | Classification du type de faille (ex : CWE-79 pour XSS) |
| **Score CVSS 3.1** | Score et vecteur (ex : `AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N`) |
| **Sévérité** | Critique / Élevée / Moyenne / Faible |
| **Phase MITRE ATT&CK associée** | Tactique correspondante (ex : Initial Access — T1190) |
| **Preuve de concept** | Commande/requête utilisée et résultat observé |
| **Impact** | Ce qu'un attaquant peut concrètement faire |

## 4. Vecteurs de test couverts par cet audit

Ce tableau récapitule les pistes de test effectivement exécutées pendant l'audit (détail des commandes dans [`01-methodologie.md`](01-methodologie.md)), avec le type de vulnérabilité qu'elles visaient à détecter. Il documente la **couverture du test**, pas des vulnérabilités confirmées — les résultats précis n'ont pas été conservés.

| Vecteur testé | Type de vulnérabilité visée | CWE | Phase MITRE ATT&CK |
|---|---|---|---|
| XSS via `search.form.php?searchText=` | Cross-Site Scripting | CWE-79 | Initial Access — T1190 |
| Injection SQL via `ticket.form.php?searchText=' OR 1=1 --` et scan sqlmap sur `incident.form.php?id=` | SQL Injection | CWE-89 | Initial Access — T1190 / Credential Access — T1552 |
| Appel direct à l'API REST (`/glpi/apirest.php/Computer`, `/glpi/apirest.php/User`) | Contrôle d'accès manquant sur l'API | CWE-306 | Reconnaissance — T1592 / Collection — T1005 |
| Accès direct à des fichiers internes (`/glpi/inc/autoload.php`, `/glpi/front/setup.php`) | Exposition d'information | CWE-200 | Reconnaissance — T1595 |
| Bruteforce du formulaire de login (Hydra + rockyou.txt) | Absence de limitation des tentatives d'authentification | CWE-307 | Credential Access — T1110 (Brute Force) |
| Permissions de `config/` et `config_db.php` | Permissions incorrectes sur ressource critique | CWE-732 | Discovery — T1083 / Credential Access — T1552.001 |
| Comparaison version GLPI vs bases CVE/Exploit-DB | Composant potentiellement non maintenu | CWE-1104 | Initial Access — T1190 |

## 5. Recommandations générales

Recommandations issues de ce type d'audit, indépendamment du résultat précis obtenu — voir le détail de remédiation par catégorie dans [`03-remediation.md`](03-remediation.md).

### Priorité haute (quick wins)
- Maintenir GLPI à jour vers la dernière version stable et appliquer les correctifs de sécurité.
- Restreindre ou authentifier systématiquement l'accès à l'API REST (`apirest.php`).
- Bloquer l'accès direct aux fichiers internes (`inc/`, `front/setup.php`) au niveau du serveur web.

### Priorité moyenne
- Politique de mot de passe fort + verrouillage de compte après échecs répétés (anti-bruteforce).
- Échapper systématiquement les entrées utilisateur pour prévenir XSS et injections SQL.
- Corriger les permissions des fichiers de configuration sensibles (`config_db.php` non lisible par des utilisateurs non autorisés).

### Priorité basse / hygiène continue
- Désactiver les services système non utilisés (FTP, Telnet...).
- Maintenir le serveur hôte à jour (patchs OS réguliers).
- Auditer périodiquement les comptes utilisateurs GLPI pour supprimer les comptes obsolètes.

## 6. Annexes

- Logs des tests d'intrusion : non conservés pour cet audit.
- Résultats des scanners (OpenVAS/Nessus) : non conservés pour cet audit.
- Captures d'écran : non incluses dans ce dépôt public.

> **Note méthodologique** : les résultats bruts de ce premier audit n'ont pas été conservés. Ce dépôt documente la méthodologie, les vecteurs testés et le format de rapport, réutilisables pour un futur audit avec conservation systématique des preuves (recommandation : centraliser logs et captures dans un dossier `evidence/` horodaté dès le prochain run).
