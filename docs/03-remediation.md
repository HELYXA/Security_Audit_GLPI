# Fiches de remédiation

Pour chaque famille de vulnérabilité **visée par les tests** menés dans cet audit (voir [`02-rapport-audit.md`](02-rapport-audit.md)), la remédiation associée et la référence au top 10 OWASP correspondant.

## Référentiel : OWASP Top 10 (2021)

| # | Catégorie OWASP | Lien avec cet audit |
|---|---|---|
| A01 | Broken Access Control | Accès non authentifié à l'API REST GLPI, manipulation d'URL vers des pages restreintes |
| A02 | Cryptographic Failures | Vérification que les mots de passe stockés (`/etc/passwd` côté OS, table utilisateurs GLPI) sont bien chiffrés/hashés |
| A03 | Injection | XSS (`search.form.php`), injection SQL (`ticket.form.php`, `incident.form.php`) |
| A05 | Security Misconfiguration | Permissions des fichiers de config, fichiers internes exposés (`inc/autoload.php`, `front/setup.php`) |
| A06 | Vulnerable and Outdated Components | Version GLPI non à jour / CVE connues |
| A07 | Identification and Authentication Failures | Absence de protection anti-bruteforce sur le login |
| A09 | Security Logging and Monitoring Failures | À vérifier : présence de logs d'authentification et d'accès API |

Référence : [owasp.org/Top10](https://owasp.org/Top10/)

---

## R-01 - Cross-Site Scripting (XSS)

**Constat** : un paramètre de requête (`searchText`) semble accepter du contenu HTML/JS non filtré.

**Remédiation** :
- Échapper systématiquement toute donnée utilisateur avant affichage (encodage contextuel HTML).
- Mettre en place une **Content-Security-Policy (CSP)** restrictive côté serveur web.
- Mettre à jour GLPI : les versions récentes corrigent les échappements manquants des versions anciennes.

## R-02 - Injection SQL

**Constat** : test de payload `' OR 1=1 --` et scan sqlmap sur des paramètres d'URL de formulaires.

**Remédiation** :
- Utiliser exclusivement des requêtes préparées / paramétrées (déjà le cas dans les versions récentes de GLPI, à vérifier sur la version testée).
- Appliquer le principe de moindre privilège sur le compte MySQL/MariaDB utilisé par GLPI (pas de droits `FILE`, `DROP`, etc. si non nécessaires).
- Mettre à jour vers la dernière version stable.

## R-03 - Accès non authentifié à l'API REST

**Constat** : les endpoints `apirest.php/User` et `apirest.php/Computer` répondent sans jeton d'authentification valide (à confirmer selon la réponse réellement observée).

**Remédiation** :
- Vérifier que l'API REST GLPI exige un `Session-Token` ou `user_token` valide sur tous les endpoints exposant des données.
- Désactiver l'API REST si elle n'est pas utilisée (`Configuration > Générale > API`).
- Restreindre l'accès réseau à l'API aux seules IP autorisées (pare-feu applicatif).

## R-04 - Exposition de fichiers internes

**Constat** : des chemins internes (`inc/autoload.php`, `front/setup.php`) sont accessibles directement en HTTP.

**Remédiation** :
- Configurer le serveur web (Apache/Nginx) pour bloquer l'accès direct aux répertoires `inc/`, `config/`.
- S'assurer que `front/setup.php` (page d'installation) est bien désactivé/supprimé après l'installation initiale, c'est une pratique standard et une faille classique si elle reste accessible.

## R-05 - Robustesse de l'authentification

**Constat** : test de bruteforce via Hydra sur le formulaire de login.

**Remédiation** :
- Implémenter un verrouillage de compte temporaire après N échecs (throttling / rate limiting).
- Imposer une politique de mot de passe fort (longueur, complexité).
- Ajouter une authentification à deux facteurs (2FA) pour les comptes à privilèges.

## R-06 - Permissions fichiers/configuration

**Constat** : vérification des permissions de `config/` et de `config_db.php`.

**Remédiation** :
- `config_db.php` ne doit être lisible que par l'utilisateur système exécutant le serveur web (`chmod 640`, propriétaire adapté).
- S'assurer que le répertoire `config/` n'est pas accessible directement via une requête HTTP.

## R-07 - Version obsolète / CVE connues

**Constat** : version GLPI à comparer aux bases CVE/Exploit-DB.

**Remédiation** :
- Mettre à jour GLPI vers la dernière version stable (voir [glpi-project.org](https://glpi-project.org/) et le [changelog officiel](https://github.com/glpi-project/glpi/releases)).
- Mettre en place un suivi régulier des CVE publiées pour GLPI (veille sécurité).

## R-08 - Hygiène du serveur hôte

**Remédiation** :
- Désactiver les services inutiles (`systemctl list-units --type=service`, puis `systemctl disable <service>`).
- Appliquer les mises à jour de sécurité du système régulièrement.
- Restreindre les accès SSH (clé uniquement, pas de root direct).
