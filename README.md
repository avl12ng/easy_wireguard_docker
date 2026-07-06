# WireGuard Easy (wg-easy) via Docker Compose

Déploiement simple et rapide d'un serveur **WireGuard** avec interface web d'administration, grâce à [wg-easy](https://github.com/wg-easy/wg-easy).

![Docker](https://img.shields.io/badge/docker-compose-blue?logo=docker)
![WireGuard](https://img.shields.io/badge/WireGuard-VPN-88171A?logo=wireguard)
![License](https://img.shields.io/badge/license-MIT-green)

🇬🇧 [English version available here](./README.en.md)

## 📋 Sommaire

- [Présentation](#-présentation)
- [Prérequis](#-prérequis)
- [Installation](#-installation)
- [Configuration](#-configuration)
- [Ouverture des ports](#-ouverture-des-ports)
- [Utilisation](#-utilisation)
- [Sécurité](#-sécurité)
- [Commandes utiles](#-commandes-utiles)
- [Dépannage](#-dépannage)
- [Licence](#-licence)

## 🎯 Présentation

Ce dépôt fournit une configuration `docker-compose` prête à l'emploi pour déployer **wg-easy**, une surcouche web pour WireGuard permettant de :

- Créer et gérer des clients VPN via une interface graphique
- Générer des QR codes de configuration pour mobile
- Superviser les connexions actives
- Administrer le tout sans ligne de commande

## ✅ Prérequis

- Un serveur Linux avec [Docker](https://docs.docker.com/engine/install/) et [Docker Compose](https://docs.docker.com/compose/install/) installés
- Une adresse IP publique fixe ou un nom de domaine (FQDN) pointant vers le serveur
- Accès administrateur à votre routeur/box pour la redirection de ports
- Le module noyau `wireguard` disponible sur l'hôte (généralement inclus dans les noyaux Linux ≥ 5.6)

## 🚀 Installation

### 1. Cloner le dépôt

```bash
git clone https://github.com/avl12ng/easy_wireguard_docker.git
cd easy_wireguard_docker
```

### 2. Générer le hash du mot de passe

L'interface web est protégée par un mot de passe. Celui-ci doit être fourni sous forme de hash **bcrypt**, généré via l'image officielle :

```bash
docker run --rm ghcr.io/wg-easy/wg-easy wgpw 'Votre_Mot_De_Passe_Fort'
```

La commande retourne une ligne du type :

```
PASSWORD_HASH='$2a$12$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
```

> ⚠️ Pensez à échapper les `$` si vous copiez ce hash directement dans un fichier `.env` en dehors de guillemets simples (voir plus bas).

### 3. Configurer le fichier `.env`

Créez un fichier `.env` à la racine du projet :

```env
WG_PUBLIC_IP=votre_ip_publique_ou_fqdn
WG_PASSWORD_HASH='$2a$12$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
```

| Variable | Description |
|---|---|
| `WG_PUBLIC_IP` | Adresse IP publique ou nom de domaine par lequel les clients VPN joindront le serveur |
| `WG_PASSWORD_HASH` | Hash bcrypt du mot de passe d'accès à l'interface web (généré à l'étape précédente) |

### 4. Lancer les conteneurs

```bash
docker compose up -d
```

Vérifiez que le conteneur démarre correctement :

```bash
docker compose logs -f wg-easy
```

## ⚙️ Configuration

Extrait du fichier `docker-compose.yml` :

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy
    container_name: wg-easy
    environment:
      - WG_HOST=${WG_PUBLIC_IP}
      - PASSWORD_HASH=${WG_PASSWORD_HASH}
      - PORT=51821
      - WG_PORT=51820
    volumes:
      - ./wg-easy-data:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

### Détail des paramètres

- **`WG_HOST`** : adresse publique annoncée aux clients (issue de `WG_PUBLIC_IP`)
- **`PASSWORD_HASH`** : hash bcrypt utilisé pour l'authentification à l'interface web
- **`PORT`** : port TCP de l'interface web d'administration (défaut `51821`)
- **`WG_PORT`** : port UDP utilisé par le protocole WireGuard (défaut `51820`)
- **`volumes`** : persistance de la configuration WireGuard et des clients dans `./wg-easy-data`
- **`cap_add`** : capacités Linux nécessaires (`NET_ADMIN` pour la gestion réseau, `SYS_MODULE` pour charger le module noyau WireGuard si besoin)
- **`sysctls`** : activation du forwarding IP, indispensable pour router le trafic VPN

## 🌐 Ouverture des ports

Sur votre routeur/box internet, redirigez le port suivant vers l'IP locale du serveur hébergeant Docker :

| Port | Protocole | Direction |
|---|---|---|
| `51820` | UDP | Entrant (obligatoire, trafic WireGuard) |
| `51821` | TCP | Entrant (optionnel, uniquement si vous administrez l'interface depuis l'extérieur) |

> 🔒 Il est recommandé de **ne pas exposer le port 51821 (interface web) directement sur internet**, mais plutôt de le limiter à un accès local ou via un reverse proxy avec authentification supplémentaire / VPN de gestion.

## 🖥️ Utilisation

1. Rendez-vous sur `http://<ip_ou_domaine>:51821`
2. Connectez-vous avec le mot de passe défini précédemment
3. Cliquez sur **New Client** pour créer un profil VPN
4. Scannez le QR code avec l'application WireGuard mobile, ou téléchargez le fichier `.conf` pour un client desktop

## 🔐 Sécurité

- Utilisez un mot de passe fort et unique pour `PASSWORD_HASH`
- Ne committez **jamais** votre fichier `.env` sur GitHub (ajoutez-le à `.gitignore`)
- Limitez l'exposition de l'interface web (port 51821) à un réseau de confiance si possible
- Pensez à activer des mises à jour régulières de l'image Docker
- Sauvegardez régulièrement le dossier `wg-easy-data` (il contient les clés privées des clients)

Exemple de `.gitignore` recommandé :

```gitignore
.env
wg-easy-data/
```

## 🛠️ Commandes utiles

```bash
# Démarrer les services
docker compose up -d

# Arrêter les services
docker compose down

# Voir les logs en direct
docker compose logs -f

# Mettre à jour l'image
docker compose pull
docker compose up -d

# Régénérer un hash de mot de passe
docker run --rm ghcr.io/wg-easy/wg-easy wgpw 'NouveauMotDePasse'
```

## 🩺 Dépannage

| Problème | Piste de résolution |
|---|---|
| Interface web inaccessible | Vérifiez que le port `51821/tcp` est bien redirigé et que le conteneur est démarré (`docker compose ps`) |
| Clients ne se connectent pas | Vérifiez la redirection du port `51820/udp` et que `WG_HOST` correspond bien à l'IP/domaine public |
| Pas de trafic internet via le VPN | Vérifiez que `net.ipv4.ip_forward=1` est bien actif et que les règles NAT/iptables de l'hôte ne bloquent pas le trafic |
| Erreur liée au module WireGuard | Assurez-vous que le noyau de l'hôte supporte WireGuard nativement, ou installez `wireguard-tools` sur l'hôte |

## 📄 Licence

Ce projet de configuration est distribué sous licence MIT. wg-easy lui-même est développé par la communauté [wg-easy](https://github.com/wg-easy/wg-easy) sous sa propre licence.

---

*Basé sur l'image officielle [ghcr.io/wg-easy/wg-easy](https://github.com/wg-easy/wg-easy)*
