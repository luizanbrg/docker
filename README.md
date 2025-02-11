# Application de vote avec Docker

Bienvenue dans la documentation de cette application de vote, un projet conçu pour illustrer l'utilisation de Docker et de l'architecture microservices. Ce projet permet aux utilisateurs de voter en temps réel et de visualiser les résultats dynamiquement à travers une interface web. 🗳️📊

## 🚀 Introduction à Docker

### 🐳 Qu'est-ce que Docker ?
Docker est une plateforme permettant de créer, déployer et exécuter des applications dans des conteneurs. Un conteneur est une unité logicielle standardisée qui regroupe tout ce dont une application a besoin pour fonctionner : code, bibliothèques, dépendances et configuration.

### 🏗️ Qu'est-ce que Docker Compose ?
Docker Compose est un outil permettant de définir et de gérer des applications multi-conteneurs. Il utilise un fichier `docker-compose.yml` pour spécifier la configuration de chaque service et facilite l'orchestration de l'ensemble des conteneurs en une seule commande.

### 🎯 Avantages de Docker
- **Portabilité** : Fonctionne de manière identique sur tous les environnements (local, cloud, production).
- **Isolation** : Chaque conteneur est indépendant, évitant les conflits entre dépendances.
- **Scalabilité** : Permet de déployer et mettre à l'échelle rapidement les services.
- **Automatisation** : Facilite le déploiement et la gestion des infrastructures.

## 🛠️ Installation et Prérequis

### Prérequis
- **Docker** et **Docker Compose** installés sur votre machine.
- **Git** pour cloner le repository.

### Installation
1. Clonez ce dépôt GitHub :
```bash
git clone git@github.com:luizanbrg/docker.git
cd docker
```
2. Configurez les variables d'environnement dans un fichier `.env` :
```env
POSTGRES_USER=votreuser
POSTGRES_PASSWORD=votremdp
POSTGRES_DB=postgres
DB_HOST=db
REDIS_HOST=redis
POSTGRES_HOST=db
POSTGRES_PORT=5432
```
3. Lancez l'application avec Docker Compose :
```bash
docker compose up --build
```

## 📦 Architecture

L'application suit une architecture microservices et repose sur plusieurs services interconnectés via Docker.

### 🏗️ Services
- **Poll (Python/Flask)** : Interface permettant aux utilisateurs de voter (port 5000), stockant temporairement les votes dans Redis.
- **Redis** : Base de données en mémoire servant de pont entre Poll et Worker.
- **Worker (Java)** : Traite les votes récupérés depuis Redis et les enregistre dans PostgreSQL.
- **PostgreSQL** : Stockage persistant des votes.
- **Result (Node.js)** : Interface affichant les résultats en temps réel (port 5001), lisant les données depuis PostgreSQL.

### 🔀 Réseaux
Trois réseaux Docker sont utilisés pour séparer les communications entre services :
- `poll-tier` : Communication entre Poll et Redis.
- `result-tier` : Communication entre Result et PostgreSQL.
- `back-tier` : Communication entre Worker, Redis et PostgreSQL.

---

## 🔗 Communication entre les services

Docker Compose facilite la communication entre les conteneurs grâce à **des réseaux internes**. Chaque service de l'application est attaché à un ou plusieurs réseaux spécifiques, permettant un échange fluide des données tout en maintenant une isolation entre certaines parties de l’application.

### 📡 Communication entre les services

1. **Le service `Poll` (Python/Flask)**
   - Exposé sur `http://localhost:5000` pour les utilisateurs.
   - Lorsqu’un utilisateur vote, l’application envoie les données à **Redis** (`redis:6379`).
   - Comme les services sont sur le même réseau `poll-tier`, le service `Poll` peut directement communiquer avec Redis en utilisant le nom `redis` (plutôt qu’une IP statique).

2. **Le service `Redis` (Stockage temporaire)**
   - Agit comme un **message broker** en stockant temporairement les votes.
   - Le `Worker` interroge Redis en continu pour récupérer les votes en attente.

3. **Le service `Worker` (Java)**
   - Récupère les votes depuis Redis (`redis:6379`).
   - Transforme et stocke ces votes dans **PostgreSQL** (`db:5432`).
   - Comme `Worker` et `PostgreSQL` sont sur le même réseau `back-tier`, il peut interagir avec PostgreSQL via `db`.

4. **Le service `PostgreSQL` (Stockage permanent)**
   - Stocke les votes envoyés par `Worker`.
   - Accessible uniquement par `Worker` et `Result`, car il est isolé sur le réseau `back-tier`.

5. **Le service `Result` (Node.js)**
   - Exposé sur `http://localhost:5001` pour afficher les résultats en temps réel.
   - Se connecte à PostgreSQL (`db:5432`) pour récupérer les votes enregistrés.
   - Comme il est dans le réseau `result-tier`, il peut communiquer avec la base de données.

---

## 🔀 Explication des réseaux
Le projet utilise **trois réseaux Docker distincts** pour organiser la communication :
| Réseau        | Connecte                      | Objectif |
|--------------|------------------------------|----------|
| `poll-tier`  | `Poll` ↔ `Redis`              | Gérer les votes entrants |
| `back-tier`  | `Worker` ↔ `Redis` ↔ `DB`     | Traiter les votes et stocker les résultats |
| `result-tier`| `Result` ↔ `DB`               | Afficher les résultats stockés |

Grâce à cette séparation, **chaque service ne voit que les autres services dont il a besoin**, réduisant ainsi les risques de sécurité et améliorant la modularité.

---

## 🛠️ Pourquoi ne pas utiliser localhost ?
Contrairement à une application classique où tout tourne sur un même serveur, Docker isole chaque service dans un **conteneur**. Ces conteneurs ont leur propre environnement réseau. 

👉 **Solution :** Docker Compose crée automatiquement des DNS internes. Plutôt que `localhost`, les services se référencent entre eux par leur **nom de service** défini dans `compose.yml` (ex : `redis`, `db`, `worker`, etc.).

Ainsi, **au lieu d'écrire :**
```python
redis = Redis(host='localhost', port=6379)
```
**Ont écrit :**
```python
redis = Redis(host='redis', port=6379)
```
Ce qui permet à `Poll` et `Worker` de retrouver **automatiquement** Redis sans configuration manuelle d’IP !

---

### 🎯 En résumé :
- Les services sont **isolés** dans des réseaux Docker distincts.
- Ils se **reconnaissent par leur nom de service** (`redis`, `db`, etc.), pas par `localhost`.
- Redis joue un rôle de **tampon** pour assurer l’acheminement des votes entre `Poll`, `Worker` et `PostgreSQL`.
- PostgreSQL stocke les votes, et `Result` les affiche en les récupérant depuis la base.

---

### 💾 Volumes
- `db-data` : Volume persistant pour stocker les données PostgreSQL.

## 📂 Structure des fichiers
```bash
voting-app/
│
├── compose.yml          # Configuration des services
├── schema.sql           # Schéma de la base de données
├── .env                 # Variables d'environnement
│
├── poll/               # Service de vote (Python)
│   └── Dockerfile
│
├── worker/            # Service de traitement (Java)
│   └── Dockerfile
│
└── result/           # Service d'affichage (Node.js)
    └── Dockerfile
```

## 📊 Utilisation
1. Accédez à l'interface de vote : `http://localhost:5000`
2. Votez pour l'une des options proposées.
3. Visualisez les résultats en temps réel sur `http://localhost:5001`.

## ⚙️ Configuration
Les services sont configurés via des variables d'environnement, garantissant flexibilité et sécurité. Docker Compose gère automatiquement les connexions entre services.

## 🔍 Améliorations possibles
- Ajout d'une authentification utilisateur.
- Développement d'une interface administrateur.
- Mise en place de monitoring des performances.
- Ajout de tests automatisés pour chaque service.

---

