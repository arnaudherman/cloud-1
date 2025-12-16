# Cloud-1 — Déploiement Automatisé d'Infrastructure Cloud

![Ansible](https://img.shields.io/badge/Ansible-2.9+-EE0000?logo=ansible&logoColor=white)
![Docker](https://img.shields.io/badge/Docker_Compose-V2-2496ED?logo=docker&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?logo=ubuntu&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)

Déploiement en une commande d'une stack WordPress prête pour la production sur infrastructure cloud publique.

---

## Architecture

```
                        ┌──────────────────┐
                        │     Internet     │
                        └────────┬─────────┘
                                 │ HTTPS (443)
                        ┌────────▼─────────┐
                        │     Traefik      │
                        │  (Reverse Proxy) │
                        │  Let's Encrypt   │
                        └────────┬─────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
     ┌────────▼────────┐ ┌───────▼───────┐ ┌───────▼───────┐
     │    WordPress    │ │  PhpMyAdmin   │ │    MariaDB    │
     │   (Port 80)     │ │   (/pma)      │ │  (Interne)    │
     └────────┬────────┘ └───────┬───────┘ └───────────────┘
              │                  │                  ▲
              └──────────────────┴──────────────────┘
                          Réseau Docker (backend)
```

**Stack :** Ansible → Docker Compose V2 → Traefik → WordPress + MariaDB + PhpMyAdmin

Tous les services tournent sur des réseaux Docker isolés. Le trafic externe arrive sur Traefik (ports 80/443), est terminé en TLS, puis routé vers les conteneurs backend. La base de données n'est jamais exposée.

---

## Choix Techniques

### Pourquoi Infomaniak ?

Fournisseur cloud européen basé sur OpenStack. Contrairement à un hébergement web classique, j'obtiens un vrai VPS avec accès root, règles firewall personnalisées et stockage bloc persistant. Conforme RGPD, pas de dépendance aux clouds US.

### Pourquoi Ansible plutôt que des scripts Bash ?

- **Idempotent** : Exécuter le playbook 10 fois donne le même résultat. Pas d'erreurs "déjà existant".
- **Agentless** : SSH uniquement, rien à installer sur la cible.
- **Déclaratif** : Je décris l'état souhaité, pas les étapes pour y arriver.
- **Secrets** : `ansible-vault` chiffre les credentials au repos avec AES-256.

### Pourquoi Traefik plutôt que Nginx ?

Nginx nécessite des mises à jour manuelles quand les conteneurs changent. Traefik surveille le socket Docker et découvre automatiquement les services via des labels. Les certificats Let's Encrypt sont générés et renouvelés automatiquement — zéro intervention manuelle.

### Sécurité : Pourquoi pas de port 8080 ?

Les installations PhpMyAdmin standard exposent le port 8080 directement. Ça contourne le TLS et expose les credentials en clair. À la place :

1. PhpMyAdmin n'a **aucun port publié**
2. Traefik route le chemin `/pma` vers le conteneur en interne
3. Tout le trafic est chiffré TLS (HTTPS uniquement)
4. Le port MariaDB 3306 n'est **jamais exposé** sur internet

---

## Structure du Projet

```
cloud-1/
├── ansible/
│   ├── ansible.cfg              # Chemin inventaire, config vault
│   ├── inventory.ini            # Serveur(s) cible(s)
│   ├── playbook.yml             # Point d'entrée principal
│   ├── group_vars/
│   │   └── all/
│   │       └── vault.yml        # Secrets chiffrés (AES-256)
│   └── roles/
│       ├── common/              # Paquets de base (git, curl, htop)
│       ├── docker/              # Docker CE + plugin Compose V2
│       └── inception/           # Déploiement stack WordPress
│           ├── tasks/main.yml
│           └── templates/
│               └── docker-compose.yml.j2
├── .gitignore
├── LICENSE
└── README.md
```

### Détail des Rôles

| Rôle | Fonction |
|------|----------|
| `common` | Mise à jour système + outils essentiels |
| `docker` | Ajoute le dépôt Docker officiel, installe `docker-compose-plugin` (V2) |
| `inception` | Template le `docker-compose.yml` avec les variables vault, déploie la stack |

**Flexibilité** : L'URL du dépôt Docker utilise `{{ ansible_distribution_release }}` pour détecter automatiquement la version Ubuntu (focal, jammy, noble). Aucune valeur codée en dur.

---

## Résilience

La stack survit aux redémarrages serveur :

- **`restart: always`** sur tous les conteneurs
- **Volumes nommés** pour la persistance des données :
  - `db_data` → Bases de données MariaDB
  - `wp_data` → Uploads/plugins WordPress
  - `traefik_data` → Certificats Let's Encrypt

---

## Installation

### Prérequis

- Machine locale : Python 3, Ansible 2.9+
- Serveur cible : Ubuntu 22.04 LTS, accès SSH
- Clé SSH : `~/.ssh/cloud1_key.pem` (ou modifier `inventory.ini`)

### Déployer

```bash
cd ansible
ansible-playbook playbook.yml --ask-vault-pass
```

C'est tout. Une seule commande.

### Accès

| Service | URL |
|---------|-----|
| WordPress | `https://votre-domaine.com` |
| PhpMyAdmin | `https://votre-domaine.com/pma` |

Les credentials par défaut sont dans le vault. Exécuter `ansible-vault view group_vars/all/vault.yml` pour les consulter.

---

## Gestion du Vault

```bash
# Voir les secrets chiffrés
ansible-vault view group_vars/all/vault.yml

# Modifier les secrets
ansible-vault edit group_vars/all/vault.yml

# Déchiffrer (déconseillé)
ansible-vault decrypt group_vars/all/vault.yml
```

---

## Vérifier la Sécurité

```bash
# MariaDB ne doit PAS répondre de l'extérieur
nc -zv <ip-serveur> 3306  # Attendu : Connection refused

# HTTPS doit fonctionner
curl -I https://votre-domaine.com  # Attendu : HTTP/2 200

# HTTP doit rediriger
curl -I http://votre-domaine.com   # Attendu : 301 -> https://
```

---

## Licence

MIT — Voir [LICENSE](LICENSE)

---

**École 42** — Projet Cloud-1
