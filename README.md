# Audit de sécurité - GLPI (labo isolé)

Projet de pentest réalisé dans un environnement de test isolé (Kali Linux attaquant une VM GLPI dédiée, réseau local `192.168.56.0/24`), dans le cadre de ma formation Bachelor Cybersécurité (alternance).

> ⚠️ **Cadre légal.** Cet audit a été mené exclusivement sur une infrastructure de laboratoire dédiée, hors ligne et sans lien avec un système en production. Toute donnée pouvant identifier l'environnement d'origine (nom de domaine, adresse IP) a été anonymisée dans ce dépôt. Aucune des techniques présentées ici n'a été utilisée sans autorisation sur un système que je ne possède pas.

## Objectif

Évaluer la sécurité d'une instance [GLPI](https://glpi-project.org/) (Gestion Libre de Parc Informatique) : configuration, surface d'attaque exposée, vulnérabilités applicatives et serveur, gestion des permissions et des accès.

## Contexte technique

| Élément | Détail |
|---|---|
| Cible | GLPI 9.x sur `glpi-lab.local` (`192.168.56.101`) |
| Machine attaquante | Kali Linux |
| Réseau | LAN isolé, VirtualBox |
| Outils | nmap, nikto, hydra, sqlmap, OpenVAS, OWASP ZAP, curl |

## Méthodologie

L'audit suit une démarche en 6 phases, détaillée dans [`docs/01-methodologie.md`](docs/01-methodologie.md) :

1. Collecte d'informations (reconnaissance)
2. Évaluation de la configuration de GLPI
3. Analyse des vulnérabilités connues (CVE)
4. Test des permissions et des accès
5. Exploitation et tests d'élévation de privilèges
6. Test de la sécurité du serveur hôte

## Rapport

Le rapport (vecteurs de test couverts, mapping CWE et MITRE ATT&CK, format de fiche de vulnérabilité, recommandations) est dans [`docs/02-rapport-audit.md`](docs/02-rapport-audit.md). Les résultats bruts de ce premier run (sorties de scan, scores CVSS précis) n'ont pas été conservés ; ce document sert de gabarit réutilisable pour un audit complet.

Les fiches de remédiation par catégorie de vulnérabilité sont dans [`docs/03-remediation.md`](docs/03-remediation.md).

## Compétences mobilisées

- Reconnaissance réseau et fingerprinting de services (nmap, banner grabbing)
- Scan de vulnérabilités web (nikto, OWASP ZAP, OpenVAS)
- Tests d'authentification par force brute (hydra)
- Tests d'injection (SQL via sqlmap, XSS manuel)
- Analyse des permissions fichiers/utilisateurs en environnement Linux
- Rédaction de rapport d'audit avec scoring de risque

## Avertissement

Ce dépôt est un travail pédagogique. Il ne contient aucune donnée réelle, aucun identifiant, aucune information permettant d'identifier un système en production.
