# MediConnect : sécurisation de l'infrastructure réseau

Projet académique réalisé en binôme à l'Efrei Paris (module Sécurité des réseaux, 2026). Il consiste à concevoir et mettre en œuvre l'architecture réseau sécurisée d'une plateforme fictive de prise de rendez-vous médicaux : segmentation, VPN, pare-feu, authentification multifacteur et journalisation.

> Maquette entièrement virtualisée (VirtualBox, Debian, Docker, pfSense CE 2.9.0). L'application web est considérée comme existante : le travail porte uniquement sur le réseau et la sécurité des accès.

## Architecture

- **2 sites**, chacun protégé par un pare-feu pfSense (FW-HQ et FW-S2), reliés par un VPN IPsec site à site.
- **6 zones de sécurité au siège**, classées par niveau de confiance : DMZ, SRV (serveurs internes), LAN, DEMO, LOGS et MGMT (administration).
- **Principe directeur** : refus par défaut. Toute communication entre zones traverse le pare-feu et n'est autorisée que pour un besoin métier explicite.
- **Zones simulées** par des réseaux internes VirtualBox et des réseaux Docker `macvlan`, pour que les conteneurs de deux zones ne communiquent qu'à travers pfSense.

## Ce qui a été mis en œuvre

| Domaine | Réalisation |
|---|---|
| VPN site à site | IPsec IKEv2, AES-256, SHA-256, DH 14, PFS ; seuls SRV et LOGS du siège sont joignables depuis le site 2 |
| VPN d'accès distant | OpenVPN (TLS 1.2 minimum, AES-256-GCM, `tls-auth`), autorité de certification interne, un certificat par utilisateur |
| Contrôle d'accès par identité | Adresse VPN fixe par groupe (support, comptabilité, employés, informaticiens, administrateurs, commerciaux) et une règle de pare-feu par groupe |
| Authentification multifacteur | Certificat + mot de passe + code TOTP, vérifiés par FreeRADIUS (PAM, `pam_google_authenticator`) |
| Filtrage | Règles par zone avec alias, refus journalisé sur chaque interface, validation « test avant / après » |
| Portail et DMZ | Reverse proxy nginx en HTTPS (TLS 1.2/1.3, HSTS, en-têtes de sécurité), espace médecins protégé par MFA |
| Contrôle applicatif | Filtrage nginx (traversée de répertoire, XSS, injection SQL), limitation de débit |
| Administration | Interface web en HTTPS uniquement, depuis la zone MGMT, comptes nominatifs, règle anti-verrouillage supprimée |
| Journalisation | Collecte syslog centralisée dans la zone LOGS, rotation `logrotate`, script d'alertes (accès non autorisé à MGMT, DEMO vers production, balayage de ports) |

## Tests

28 tests documentés (résultat attendu contre résultat obtenu), couvrant Internet, le site 2, les 6 profils VPN, la MFA, le filtrage applicatif, chaque zone, l'administration et la détection de scan. Exemples :
- depuis Internet, seul le portail est accessible ; les autres ports sont filtrés et les adresses internes injoignables ;
- sans code TOTP, la connexion VPN est refusée (`AUTH_FAILED`) ; avec un code valide, le tunnel s'établit ;
- avant durcissement, le LAN atteignait l'interface du pare-feu (200) ; après, il ne l'atteint plus (000) ;
- 100 requêtes simultanées sur le portail : 25 servies, 75 limitées (503).

## Difficultés rencontrées

Pings entre zones qui ne passaient que pendant une capture (mode promiscuité), option « Invert match » oubliée sur une règle, code TOTP consommé deux fois par les sous-requêtes nginx (résolu par un cache de 5 minutes), décalage d'horodatage pfSense dans le script d'alertes, comptes ajoutés par erreur au groupe administrateurs (détectés et corrigés).

## Limites assumées

- Internet est simulé ; serveurs et postes sont des conteneurs ; pas de NTP externe.
- Pas d'IPS (Suricata non installable sans accès Internet) ni de WAF complet.
- Pas de haute disponibilité : chaque pare-feu est un point de défaillance unique.
- Autorité de certification auto-signée et mots de passe de test.
- Authentification des médecins en HTTP basic avec cache, à remplacer par des sessions sécurisées.
- Pas de SIEM, de sauvegarde ni de chiffrement des données au repos.
- Des données de santé réelles exigeraient un hébergement certifié HDS.

## Compétences mises en œuvre

pfSense · IPsec · OpenVPN · PKI · FreeRADIUS · TOTP · segmentation réseau · DMZ · nginx · Docker · rsyslog · tcpdump · tests de sécurité · documentation technique
