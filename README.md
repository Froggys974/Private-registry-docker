# 📦 Docker Registry Privé avec TLS et Authentification

Ce projet déploie un **registre Docker privé sécurisé**, avec :

- 🔐 Chiffrement TLS (HTTPS)
- 👤 Authentification par mot de passe (via htpasswd)
- 💾 Persistance des images Docker
- 🔍 API REST du registre Docker

---

## 🛠️ Prérequis

- Docker + Docker Compose
- `openssl` (pour générer les certificats auto-signés)
- Aucun outil supplémentaire requis sur la machine hôte : `htpasswd` est généré via un conteneur Docker.

---

## 📁 Structure attendue

```

project-root/
│
├── docker-compose.yml
├── .env                         # Variables d'environnement (TLS et auth)
├── data/                        # Volume persistant du registre
└── secrets/                     # Certificats et htpasswd
  ├── registryfgrdn.cert       # Certificat TLS
  ├── registryfgrdn.key        # Clé privée TLS
  └── htpasswd                 # Fichier d'authentification

````

---

## 🔐 1. Générer les certificats TLS (auto-signés)

Le registre s'exécutera sous le nom `registry.fgrdn.fr`. Tu dois générer un certificat avec ce nom dans le **Common Name (CN)** :

```bash
mkdir -p secrets

openssl req -newkey rsa:4096 -nodes -sha256 -keyout secrets/registryfgrdn.key \
  -x509 -days 365 -out secrets/registryfgrdn.cert \
  -subj "/C=FR/ST=France/L=Paris/O=Registry/CN=registry.fgrdn.fr"
````

---

## 🖥️ 2. Configurer le client Docker pour utiliser ce certificat

Sur **chaque machine cliente** (y compris celle qui exécute Docker), installe le certificat pour que Docker puisse se connecter au registre sécurisé :

```bash
sudo mkdir -p /etc/docker/certs.d/registry.fgrdn.fr:5000

sudo cp secrets/registryfgrdn.cert /etc/docker/certs.d/registry.fgrdn.fr:5000/ca.crt
```

> ⚠️ Le fichier doit s’appeler **`ca.crt`** exactement.

---

## 🧭 3. Ajouter une entrée DNS locale (via `/etc/hosts`)

Ajoute dans ton `/etc/hosts` la ligne suivante **en remplaçant `192.168.1.107` par l’adresse IP locale de la machine qui héberge le registre** :

```text
# IP locale du serveur (trouvée avec `ip a`)
192.168.1.xxx registry.fgrdn.fr
127.0.0.1 registry.fgrdn.fr
```

Pour connaître ton IP :

```bash
ip a
```

---

## 👤 4. Générer le fichier `htpasswd` avec Docker

```bash
docker run --rm --entrypoint htpasswd httpd:2 -Bbn admin MonSuperMotDePasse > secrets/htpasswd
```

---

## ⚙️ 5. Créer le fichier `.env` si pas déjà créé

```env
REGISTRY_HTTP_TLS_CERTIFICATE=/run/secrets/registry_crt
REGISTRY_HTTP_TLS_KEY=/run/secrets/registry_key
REGISTRY_AUTH=htpasswd
REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm
```

---

## 🚀 6. Lancer le registre

```bash
docker compose up -d
```

---

## 🐳 7. Utilisation du registre

### 🔐 Connexion

```bash
docker login https://registry.fgrdn.fr:5000
username: admin
password: MonSuperMotDePasse
```

### 📤 Pusher une image

```bash
docker tag frontend registry.fgrdn.fr:5000/frontend:1.0.0
docker push registry.fgrdn.fr:5000/frontend:1.0.0
```

### 📥 Télécharger une image

```bash
docker pull registry.fgrdn.fr:5000/frontend1.0.0
```

### 🚪 Déconnexion

```bash
docker logout https://registry.fgrdn.fr:5000
```

---

## 🔍 8. Explorer les images avec l'API REST

### 📚 Lister les images disponibles

```bash
curl -u admin:MonSuperMotDePasse https://registry.fgrdn.fr:5000/v2/_catalog
```

### 🏷️ Lister les tags d'une image

```bash
curl -u admin:MonSuperMotDePasse https://registry.fgrdn.fr:5000/v2/nginx/tags/list
```

---

## 🛑 9. Arrêter et nettoyer

```bash
docker compose down
```

Supprimer les données (facultatif) :

```bash
rm -rf data/
```

---

## 🔒 Sécurité

* Ce registre utilise un certificat **auto-signé**. En production, utilisez un certificat signé par une autorité.
* Ne versionnez jamais :

  * Le dossier `secrets/`
  * Le fichier `.env`
* À ajouter dans `.gitignore` :

```bash
# .gitignore
data/
secrets/
.env
```

---

## 📚 Références

* [Docker Registry Documentation](https://docs.docker.com/registry/)
* [Docker Registry HTTP API V2](https://docs.docker.com/registry/spec/api/)
