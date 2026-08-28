# Détection de vulnérabilités web avec OWASP ZAP — Home Lab

Projet pratique de découverte de la détection de vulnérabilités web avec **OWASP ZAP** (Zed Attack Proxy), testé contre **DVWA** (Damn Vulnerable Web Application), dans un home lab virtualisé sous contrainte de ressources limitées.

## Objectif

Mettre en place une chaîne complète d'analyse de vulnérabilités web :
1. Installation et configuration de ZAP
2. Mise en place d'une cible volontairement vulnérable (DVWA)
3. Scan passif (analyse du trafic capturé)
4. Scan actif (tests d'attaque automatisés)
5. Analyse des résultats et génération d'un rapport de vulnérabilités

## Compétences démontrées

- **Installation et configuration d'outils de sécurité** (OWASP ZAP) sur un environnement Linux à ressources contraintes
- **Déploiement et administration d'une cible applicative vulnérable** (DVWA) via systemd, MariaDB et nginx
- **Méthodologie de test d'intrusion web** : reconnaissance passive (analyse de trafic proxifié), scan actif (tests d'attaque automatisés), analyse et priorisation des risques
- **Diagnostic et résolution d'incidents systèmes réels** rencontrés en conditions réelles : timeout systemd au démarrage, dépendances de services mal ordonnées, blocage d'authentification polkit, erreurs de configuration proxy
- **Lecture critique de rapports de sécurité** : identification, catégorisation par sévérité et compréhension des remédiations pour chaque vulnérabilité détectée
- **Rédaction de documentation technique structurée**, reproductible et versionnée
- **Gestion de version et publication avec Git/GitHub**

## Environnement

| Composant | Détail |
|---|---|
| Hyperviseur | Oracle VirtualBox |
| Système invité | Kali Linux |
| Ressources allouées | ~4 Go RAM / 2 CPU |
| Outil de scan | OWASP ZAP 2.17.0 |
| Cible | DVWA (paquet natif Kali, `apt install dvwa`) |

Ce projet a été volontairement réalisé sur une configuration à ressources limitées, ce qui a nécessité d'adapter certains réglages (politique de scan, gestion des services) par rapport à une installation par défaut — voir la [feuille de route](docs/feuille-de-route-zap.md) pour le détail des choix techniques et leur justification.

## Structure du repo

```
.
├── README.md
├── docs/
│   └── feuille-de-route-zap.md   # Déroulé complet du projet, étape par étape
├── screenshots/                  # Captures d'écran des étapes clés
└── reports/                      # Rapport de scan ZAP (généré en fin de projet)
```

## Problèmes rencontrés et résolus

Ce lab n'a pas été un long fleuve tranquille — plusieurs incidents réels rencontrés et leur résolution :

- **`dvwa.service` en timeout au démarrage** : causé par MariaDB non démarrée en amont (service `disabled` par défaut), dont l'initialisation était trop lente pour le délai de démarrage du service DVWA sur une machine à ressources limitées. Résolu en démarrant MariaDB séparément avant DVWA.
- **Fenêtre d'authentification polkit bloquante** au lancement de `dvwa-start` en graphique : contournée en lançant le service directement via `sudo systemctl start dvwa.service` en ligne de commande.
- **Erreur de connexion ZAP → URL malformée** (`https://http//127.0.0.1:42001`) : erreur de saisie dans le champ URL de ZAP, corrigée en retapant l'adresse proprement.
- **Migration de la VM** d'un disque à un autre (arrêt propre, pas de sauvegarde d'état, utilisation de la fonction native `Déplacer...` de VirtualBox pour préserver la configuration).

Le détail complet, commande par commande, est dans la [feuille de route](docs/feuille-de-route-zap.md).

## Captures d'écran

| Étape | Capture |
|---|---|
| Interface ZAP — Démarrage rapide | `screenshots/01-zap-demarrage-rapide.png` |
| Lancement du navigateur proxifié (Manual Explore) | `screenshots/02-manual-explore-lancer-navigateur.png` |
| Incident : authentification polkit bloquante | `screenshots/03-dvwa-authentification-polkit.png` |

## Rapport de scan

*(À venir — le rapport final ZAP sera ajouté dans `reports/` une fois le scan actif terminé.)*

## Avertissement

Ce projet est réalisé exclusivement dans un environnement local isolé (VM), contre **DVWA**, une application volontairement vulnérable conçue pour l'apprentissage de la sécurité. Aucun système tiers n'a été scanné. Ne jamais utiliser ces techniques contre des systèmes sans autorisation explicite.

## Références

- [OWASP ZAP](https://www.zaproxy.org/)
- [DVWA](https://github.com/digininja/DVWA)
