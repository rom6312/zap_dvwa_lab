# Feuille de route — Détection de vulnérabilités web avec OWASP ZAP
### Environnement : Kali Linux · 4 Go RAM · 2 CPU

> Ce document reprend l'intégralité du Projet 4, adapté pour un home lab à ressources limitées. Coche les cases au fur et à mesure, et reviens sur les sections déjà faites en cas de problème.

---

## Suivi d'avancement

- [x] Exercice 1 — Installer ZAP
- [x] Exercice 2 — Configurer le proxy
- [x] Exercice 3 — Installer DVWA + scan passif
- [x] Exercice 4 — Scan actif *(fait, mais à relancer avec un scope restreint — voir note dans la section 4)*
- [x] Exercice 5 — Analyse des résultats + rapport *(rapport généré le 12/08/2026)*

---

## 🔎 Vérifier l'état actuel avant de reprendre le travail

Contrairement à l'installation (`apt install`), qui reste en place après un redémarrage de la VM, **certains services doivent être relancés manuellement à chaque session** parce qu'ils ne sont pas configurés en démarrage automatique. Avant de reprendre là où tu t'es arrêté, vérifie dans cet ordre :

### 1. Ressources disponibles
```bash
free -h
df -h /
```

### 2. MariaDB (base de données de DVWA)
```bash
sudo systemctl status mariadb
```
Cherche `Active: active (running)`. Si tu vois `inactive (dead)` :
```bash
sudo systemctl start mariadb
sleep 5
```

### 3. DVWA
```bash
sudo systemctl status dvwa.service
```
Cherche `Active: active (running)`. Si ce n'est pas le cas — **et seulement après avoir confirmé que MariaDB tourne (étape 2)** :
```bash
sudo systemctl start dvwa.service
```
⚠️ Ne pas utiliser `dvwa-start` seul si MariaDB vient d'être arrêtée — c'est cet enchaînement précis (MariaDB d'abord, avec un court délai) qui a résolu le timeout rencontré précédemment.

### 4. Accès web à DVWA
Ouvre `http://127.0.0.1:42001` dans le navigateur **lancé depuis ZAP** (Démarrage rapide > Explorer manuellement > Lancer le navigateur). Si la page de login s'affiche, DVWA est opérationnelle.

### 5. ZAP
```bash
zaproxy -version
```
Confirme juste que le paquet est toujours présent (il ne se désinstalle jamais tout seul). Relance ensuite `zaproxy` si l'interface n'est pas déjà ouverte.

Si les 4 points ci-dessus sont bons, tu peux reprendre directement où tu en étais — pas besoin de tout réinstaller.

---

## ⚠️ Note sur la langue de l'interface

Ton ZAP est en français. Les captures/docs officielles restent en anglais dans la majorité des tutoriels — **les noms de menus ci-dessous sont donnés en anglais (référence officielle)**, avec leur équivalent français probable entre parenthèses. Si le libellé exact ne correspond pas chez toi, cherche par position dans le menu (la structure ne change pas) plutôt que par le texte exact.

---

## Exercice 1 — Installer ZAP ✅ Fait

### Ce qui a été vérifié et exécuté
```bash
free -h
nproc
df -h /
zaproxy -version        # vérifier si déjà présent
sudo apt update
sudo apt upgrade -y
sudo apt install zaproxy -y
owasp-zap -h             # vérifier la RAM détectée et le heap Java (-Xmx) attribué automatiquement
zaproxy                  # premier lancement
```

**Résultat** : interface ZAP installée et fonctionnelle, confirmée par toi.

**Point retenu pour la suite** : avec 4 Go de RAM, le heap Java alloué automatiquement à ZAP tourne autour de 700–900 Mo. À surveiller quand DVWA + navigateur tourneront en même temps.

---

## Exercice 2 — Configurer le proxy ✅ Fait

### Étapes réalisées
1. Vérification de la RAM disponible avec ZAP ouvert (`free -h`)
2. Vérification du port d'écoute local dans **Tools > Options > Network > Local Servers/Proxies** *(Outils > Options > Réseau > Serveurs locaux/Proxies)* — port par défaut `8080`
3. Lancement du navigateur **depuis ZAP** (Quick Start / *Démarrage rapide* → **Manual Explore** / *Explorer manuellement* → **Launch Browser** / *Lancer le navigateur*) plutôt qu'une config manuelle du navigateur système : ZAP configure automatiquement le proxy et installe son certificat racine, ce qui évite les erreurs de certificat HTTPS et les manipulations supplémentaires — plus économe en RAM que d'ajouter une extension type FoxyProxy à un navigateur déjà installé.
4. Vérification du trafic capturé dans l'onglet **Sites**

**Résultat** : proxy actif et confirmé, trafic visible dans l'onglet Sites.

---

## Exercice 3 — Installer une cible (DVWA) et faire un scan passif

### Pourquoi DVWA via `apt` et pas via Docker

Le projet original propose souvent Docker pour DVWA. Sur ta machine (4 Go / 2 CPU), ce n'est **pas le choix le plus sûr ni le plus léger** :
- Le démon Docker à lui seul consomme 200–400 Mo de RAM en permanence, même sans conteneur actif.
- L'image Docker `vulnerables/web-dvwa`, la plus utilisée, est basée sur **Debian 9.2** et n'a pas été mise à jour depuis plusieurs années (paquets EOL, aucune maintenance de sécurité).
- Kali propose désormais **DVWA en paquet natif dans ses dépôts officiels**, packagé et maintenu par l'équipe Kali elle-même (nginx + MariaDB + PHP 8.4), avec un service systemd propre à démarrer/arrêter à la demande. C'est plus léger, plus à jour, et plus simple à couper quand tu ne l'utilises pas — donc plus adapté à tes contraintes.

### 3.1 Vérifier la RAM disponible avant d'ajouter DVWA
```bash
free -h
```

### 3.2 Installer DVWA
```bash
sudo apt update
sudo apt install dvwa -y
```
Taille d'installation : environ 1.7 Mo (les dépendances nginx/mariadb-server/php8.4 sont probablement déjà partiellement présentes sur Kali).

### 3.3 Démarrer le service DVWA
```bash
sudo systemctl start mariadb
sleep 5
sudo systemctl start dvwa.service
```
**Incident réel rencontré** : `dvwa-start` seul échouait avec un timeout ("Failed to start dvwa.service: Connexion terminée par expiration du délai d'attente"). Cause : MariaDB (`disabled` par défaut sur Kali) n'a pas le temps de terminer son initialisation dans le délai imparti à `dvwa.service`. Solution : démarrer MariaDB séparément d'abord (elle n'a pas ce timeout serré), attendre qu'elle soit `active (running)`, puis démarrer `dvwa.service`.

**Second incident** : lancer `dvwa-start` graphiquement déclenchait une fenêtre d'authentification polkit qui refusait le mot de passe. Solution : utiliser directement `sudo systemctl start dvwa.service` en ligne de commande, qui passe par l'agent `sudo` du terminal (déjà authentifié) plutôt que par l'agent graphique polkit.

Sortie attendue de `sudo systemctl status dvwa.service` : `Active: active (running)`, avec le process nginx démarré. Accès ensuite via `http://127.0.0.1:42001`, identifiants par défaut `admin` / `password`.

### 3.4 Se connecter et initialiser la base
1. Ouvre `http://127.0.0.1:42001` dans le navigateur **lancé depuis ZAP** (pour que le trafic soit proxifié dès le départ)
2. Connecte-toi avec `admin` / `password`
3. Clique sur **Create / Reset Database** pour initialiser la base de données de l'application

### 3.5 Réaliser le scan passif
1. Navigue dans les différentes sections de DVWA (SQL Injection, XSS, etc.) pendant quelques minutes, en cliquant sur les liens du menu — c'est cette navigation qui génère le trafic
2. Reviens dans ZAP, onglet **Sites** : l'arborescence de `127.0.0.1:42001` doit apparaître avec les pages visitées
3. Vérifie l'onglet **Alerts** *(Alertes)* : le scanner passif de ZAP analyse en continu ce trafic et remonte déjà de premières observations (en-têtes manquants, cookies mal configurés, etc.), sans avoir encore attaqué activement l'application

### Gestion RAM pour cette étape
Quand tu as fini de naviguer et avant de passer à l'exercice 4, tu peux libérer de la mémoire si besoin :
```bash
free -h        # vérifier l'état
# dvwa-stop    # à utiliser seulement si tu dois couper DVWA temporairement — sinon laisse tourner pour l'exercice 4
```

---

## Exercice 4 — Scan actif

Le scan actif est la partie la plus gourmande en CPU/RAM du projet : ZAP envoie ici de vraies requêtes d'attaque (injections, fuzzing de paramètres, etc.). Sur 2 CPU / 4 Go, on va délibérément **réduire l'intensité** plutôt que d'utiliser les réglages par défaut.

### 4.1 Vérifier les ressources juste avant de lancer
```bash
free -h
```
Idéalement, garde un onglet terminal ouvert pendant le scan pour surveiller :
```bash
watch -n 5 free -h
```

### 4.2 Restreindre le scope à ta seule cible (important)

**Incident réel rencontré** : le premier scan actif a touché des domaines externes réels (`google.com`, `gstatic.com`, `contrib.rocks`) en plus de `127.0.0.1:42001`, car ces ressources sont chargées par DVWA (reCAPTCHA, badge de contributeurs) et le scope n'était pas restreint. Attaquer activement des domaines tiers, même de façon marginale, n'est pas une bonne pratique — même si ces domaines appartiennent à des infrastructures robustes qui ne craignent rien.

**Avant de relancer un scan actif**, restreins le scope :
1. Clic droit sur `127.0.0.1:42001` dans l'onglet **Sites**
2. **Include in Context > Default Context** *(Inclure dans le contexte > Contexte par défaut)* si ce n'est pas déjà fait
3. Dans **Analyze > Scope** *(Analyser > Portée)*, ajoute uniquement `127.0.0.1:42001` comme cible en scope
4. Vérifie dans les options du scan actif que l'option "Scan only in scope" / *"Analyser uniquement dans la portée"* est activée

### 4.3 Réduire la charge du scan (avant de lancer)
Dans ZAP : **Tools > Options > Active Scan** *(Outils > Options > Analyse active)*
- **Concurrent scanning threads per host** *(Threads simultanés par hôte)* : valeur par défaut = 2 sur une machine à 2 cœurs, c'est déjà raisonnable — ne pas augmenter
- **Max concurrent scanned hosts** *(Hôtes scannés simultanément)* : laisser à 1, tu ne cibles qu'une seule application

Dans **Analyze > Scan Policy Manager** *(Analyser > Gestionnaire de politique de scan)`, sur la politique par défaut :
- Réduis l'**Attack Strength** *(Force de l'attaque)* de "High" à **"Medium"** ou **"Low"** pour limiter le nombre de requêtes envoyées
- Laisse l'**Alert Threshold** *(Seuil d'alerte)* sur "Medium" (par défaut) pour ne pas noyer les résultats sous les faux positifs

### 4.4 Lancer le scan actif
1. Clic droit sur le site cible dans l'onglet **Sites**
2. **Attack > Active Scan** *(Attaque > Analyse active)*
3. Sélectionne la politique ajustée à l'étape précédente
4. Démarre le scan

### 4.5 Suivre le scan
Onglet **Active Scan** *(Analyse active)* en bas de l'interface : barre de progression et nombre de requêtes envoyées. Sur ta config, prévois que ce soit plus long que sur une machine standard — c'est normal et voulu (on a volontairement ralenti pour ne pas saturer les 4 Go).

**Si le système devient très lent ou que le swap s'active fortement** (`free -h` montre "Swap" utilisé en grande quantité) : mets le scan en pause (bouton pause dans l'onglet Active Scan), ferme les autres applications ouvertes, puis reprends.

---

## Exercice 5 — Analyser les résultats et générer le rapport

### 5.1 Consulter les alertes
1. Onglet **Alerts** *(Alertes)*
2. Les vulnérabilités sont classées par sévérité (High / Medium / Low / Informational)
3. Clique sur chaque alerte pour voir : description, niveau de risque, requête/réponse concernée, et les recommandations de remédiation associées

### 5.2 Générer le rapport
1. Menu **Report > Generate Report...** *(Rapport > Générer un rapport...)*
2. Choisis un template — pour un premier rapport lisible : **"Risk and Confidence HTML"** ou **"Modern HTML"**
3. Sélectionne le(s) site(s) à inclure (ici, `127.0.0.1:42001`)
4. Choisis le dossier de destination et génère

Le rapport HTML généré contient la synthèse des risques, le détail de chaque alerte et les recommandations — c'est le livrable final du projet.

---

## Annexe A — Dépannage courant

| Problème | Cause probable | Solution |
|---|---|---|
| Rien n'apparaît dans l'onglet Sites | Navigateur non proxifié par ZAP | Relancer le navigateur depuis Quick Start > Manual Explore, ne pas utiliser le navigateur système |
| Erreur de certificat HTTPS | Navigateur système utilisé au lieu de celui lancé par ZAP | Utiliser uniquement le navigateur lancé depuis ZAP, qui installe automatiquement le certificat racine |
| Système très lent pendant le scan actif | RAM saturée (4 Go partagés entre ZAP + DVWA + navigateur) | Réduire l'Attack Strength à Low, fermer les apps inutiles, surveiller avec `free -h` |
| `dvwa-start` échoue | Port 42001 déjà utilisé, ou service mal initialisé | `dvwa-stop` puis `dvwa-start` à nouveau ; vérifier `sudo systemctl status dvwa.service` |
| ZAP ne trouve pas le navigateur | Chemin du binaire navigateur non détecté | `Tools > Options > Selenium > Binaries` pour préciser le chemin manuellement |

## Annexe B — Bonnes pratiques (rappel sécurité)

- DVWA ne doit **jamais** être exposé au-delà de `127.0.0.1` / du réseau local isolé de ton home lab
- N'utilise ZAP en scan actif **que** sur des applications que tu es autorisé à tester (ici DVWA, conçu pour ça)
- Pense à faire `dvwa-stop` quand tu ne t'en sers pas, pour libérer de la RAM et ne pas laisser tourner une appli volontairement vulnérable en continu
