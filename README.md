# E7 — MITRE ATT&CK : Reconstruction d'une Chaîne d'Attaque Complète
 
> Rapport de synthèse final de l'investigation **Buttercup Games**. Cet article relie les 6 épisodes précédents en une kill chain complète, mappée sur le framework MITRE ATT&CK, et constitue le rapport d'escalade transmis à l'équipe N2.
 
---
 
## 📚 Table des matières
 
- [1. Contexte global](#1-contexte-global)
- [2. Kill Chain complète](#2-kill-chain-complète)
- [3. Détail des techniques ATT&CK](#3-détail-des-techniques-attck)
- [4. Gaps d'investigation](#4-gaps-dinvestigation)
- [5. Rapport d'escalade — Éléments transmis à l'équipe N2](#5-rapport-descalade--éléments-transmis-à-léquipe-n2)
- [6. Conclusion](#6-conclusion)
---
 
## 1. Contexte global
 
L'équipe SOC de **Buttercup Games** a détecté et investigué une série d'attaques coordonnées contre l'infrastructure de la société. L'investigation a débuté par la détection d'un email de phishing et s'est conclue par la découverte d'un webshell actif sur le serveur web.
 
**Timeline des incidents :**
 
| Épisode | Outil | Incident détecté |
|---------|-------|-----------------|
| E1 | Email | Tentative de phishing usurpant Proton for Business |
| E2 | Splunk | Reconnaissance web — IP sondant `/passwords.pdf` |
| E3 | Splunk | Analyse comportementale — double activité suspecte |
| E4 | Splunk | Dashboard SOC + alerte automatique configurée |
| E5 | Wireshark | Brute force FTP — compte `jenny` compromis |
| E6 | Wireshark | Post-exploitation — webshell `shell.php` déployé |
 
> ⚠️ **L'investigation révèle un attaquant persistant et méthodique.** Face à chaque obstacle, il a changé de tactique — du phishing à la reconnaissance web, puis au brute force FTP.
 
---
 
## 2. Kill Chain complète
 
La progression de l'attaque suit la kill chain de Lockheed Martin et se mappe directement sur le framework MITRE ATT&CK :
 
| Phase | Technique MITRE | ID | Outil | Résultat | Épisode |
|-------|----------------|-----|-------|---------|---------|
| Initial Access | Phishing | T1566 | Email | ❌ Échec — employé non piégé | [E1](https://github.com/Paulcyber06/Analyse-de-Phishing-Usurpation-de-la-Marque-Proton) |
| Discovery | File and Directory Discovery | T1083 | Splunk | ❌ Échec — `/passwords.pdf` non trouvé | [E2](https://github.com/Paulcyber06/E1-SPL-Basics-Splunk-reconnaissance-Detection) |
| Discovery | Network Service Discovery | T1046 | Splunk | ⚠️ Serveur web cartographié | [E3](https://github.com/Paulcyber06/E2-SPL-Basics-Splunk-Transforming-Commands) |
| Credential Access | Brute Force | T1110 | Wireshark | ❌ Credentials compromis : jenny/password123 | [E5](https://github.com/Paulcyber06/E1-Wireshark-FTP-Brute-Force-Detection) |
| Lateral Movement | Valid Accounts | T1078 | Wireshark | ❌ Connexion FTP réussie | [E5](https://github.com/Paulcyber06/E1-Wireshark-FTP-Brute-Force-Detection) |
| Persistence | Web Shell | T1505.003 | Wireshark | ❌ shell.php déployé dans /var/www/html | [E6](https://github.com/Paulcyber06/E2-Wireshark-FTP-Post-Exploitation) |
| Execution | Exploit Public-Facing Application | T1190 | Wireshark | ❌ Webshell accédé via navigateur | [E6](https://github.com/Paulcyber06/E2-Wireshark-FTP-Post-Exploitation) |
 
---
 
## 3. Détail des techniques ATT&CK
 
### T1566 — Phishing
**Épisode 1** — L'attaquant, ayant eu connaissance que Buttercup Games utilise **Proton for Business**, a usurpé l'identité de Proton pour tenter de voler les credentials d'un employé via une fausse page de connexion hébergée sur `vercel.app`.
 
- SPF : ❌ fail
- DMARC : ❌ fail
- DKIM : ⚠️ pass — mais signé par `bttlazer.org` (domaine attaquant)
---
 
### T1083 — File and Directory Discovery
**Épisodes 2 & 3** — L'IP `87.194.216.51` a sondé activement le serveur web à la recherche de fichiers sensibles sur une période de **7 jours**. La cible principale : `/passwords.pdf`, tentée **3 fois** à des dates différentes.
 
- Technique : **slow and low** — basse fréquence pour éviter la détection
- Double activité détectée : 894 accès légitimes (status 200) simultanés aux tentatives de reconnaissance
---
 
### T1110 — Brute Force
**Épisode 5** — N'ayant pas trouvé `/passwords.pdf`, l'attaquant a lancé une attaque brute force FTP contre le compte `jenny`.
 
| Élément | Valeur |
|---------|--------|
| Date | 2021-02-01 à 23:26:22 |
| Credentials trouvés | jenny / password123 |
| Durée de l'attaque | Quelques secondes |
| Wordlist probable | rockyou.txt |
 
---
 
### T1078 — Valid Accounts
**Épisode 5** — Après avoir trouvé les credentials, l'attaquant s'est connecté au serveur FTP avec le compte `jenny` et a immédiatement commencé la phase de post-exploitation.
 
---
 
### T1505.003 — Web Shell
**Épisode 6** — En moins de 15 secondes après la connexion FTP, l'attaquant a :
 
1. Identifié le système : **Linux UNIX Type L8**
2. Localisé la racine web : `/var/www/html`
3. Uploadé : `shell.php`
4. Accordé les permissions : `CHMOD 777`
5. Accédé au webshell via navigateur : `GET /shell.php` à **23:26:58**
---
 
### T1190 — Exploit Public-Facing Application
**Épisode 6** — L'accès au webshell a été confirmé via la requête HTTP :
 
```
GET /shell.php HTTP/1.1
Host: 192.168.0.115
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Firefox/78.0
```
 
L'attaquant utilise une **machine Linux** avec Firefox 78.0.
 
---
 
## 4. Gaps d'investigation
 
En tant qu'analyste SOC L1, certaines questions restent sans réponse et nécessitent une investigation approfondie par l'équipe N2 :
 
| Question | Hypothèse | Priorité |
|----------|-----------|---------|
| Comment l'attaquant a-t-il obtenu le nom d'utilisateur `jenny` ? | Ancien employé / connaissance interne / document obtenu en amont non détecté | 🔴 Critique |
| L'IP `87.194.216.51` et l'attaquant FTP sont-ils la même personne ? | Mode opératoire similaire — méthodique et persistant | 🟡 À confirmer |
| Le webshell `shell.php` a-t-il été utilisé après le premier accès ? | Non déterminé — nécessite analyse des logs serveur web | 🔴 Critique |
| D'autres comptes ont-ils été ciblés ? | Non déterminé — nécessite audit complet des logs FTP | 🟡 À vérifier |
 
---
 
## 5. Rapport d'escalade — Éléments transmis à l'équipe N2
 
### 🔴 Actions immédiates requises
 
- Supprimer `/var/www/html/shell.php` du serveur
- Désactiver le compte `jenny` et forcer la réinitialisation du mot de passe
- Bloquer l'IP `192.168.0.115` et `87.194.216.51` au niveau du pare-feu
- Isoler le serveur `192.168.0.115` le temps de l'investigation
### 📋 Éléments de preuve collectés
 
| Élément | Source | Épisode |
|---------|--------|---------|
| Email de phishing + IOCs | Analyse email | E1 |
| IP suspecte `87.194.216.51` | Logs Splunk | E2/E3 |
| Dashboard SOC actif | Splunk | E4 |
| Credentials compromis jenny/password123 | Capture Wireshark | E5 |
| Webshell shell.php dans /var/www/html | Capture Wireshark | E6 |
| OS attaquant : Linux x86_64, Firefox 78.0 | Capture Wireshark | E6 |
| Heure de compromission : 2021-02-01 23:26:31 | Capture Wireshark | E5/E6 |
 
### 🔍 Investigation complémentaire demandée
 
- Analyser les logs Apache pour détecter tout accès à `shell.php` après le premier
- Vérifier les logs FTP pour identifier d'autres tentatives de brute force
- Corréler l'IP `87.194.216.51` avec l'IP source de l'attaque FTP
- Investiguer l'origine du nom d'utilisateur `jenny`
---
 
## 6. Conclusion
 
> 🔴 **Buttercup Games a été compromise. Un webshell actif est présent sur le serveur web.**
 
Cette investigation illustre une attaque en **trois phases distinctes** :
 
1. **Tentative d'accès social** (E1) — Phishing Proton → échec
2. **Reconnaissance et cartographie** (E2/E3/E4) — Splunk détecte l'IP qui cherche `/passwords.pdf` → échec
3. **Accès direct par force brute** (E5/E6) — Brute force FTP → compromission totale en 32 secondes
**Ce que cette investigation démontre :**
 
Un attaquant déterminé ne s'arrête pas à un premier échec. Il adapte sa tactique jusqu'à trouver le maillon faible — ici, un mot de passe faible sur un protocole non chiffré.
 
**Ce que le SOC a mis en place :**
- Dashboard de surveillance automatique (E4)
- Alertes en temps réel sur les comportements suspects
- Documentation complète pour l'escalade N2
---
 ## 📁 Reproduire cette analyse

Ce rapport est basé sur les investigations des épisodes E1 à E6.
Tous les fichiers sources sont disponibles dans leurs articles respectifs :

- **E1** — Email de phishing réel (non distribué pour des raisons de confidentialité)
- **E2/E3/E4** — [tutorialdata.zip Splunk](https://docs.splunk.com/images/Tutorial/tutorialdata.zip)
- **E5/E6** — [TryHackMe — room h4cked](https://tryhackme.com/room/h4cked)
