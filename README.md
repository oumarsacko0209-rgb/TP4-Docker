# TP4 : Docker et Docker Compose (application MERN)

## 1. Objectif
L'objectif de ce TP est d'apprendre à containeriser une application avec Docker, de comprendre à quoi sert Docker Compose pour orchestrer plusieurs conteneurs, et de savoir déployer une application MERN (MongoDB, Express, React, Node.js) avec Docker.

## 2. Structure du projet
TP4-Docker/
├── client/              # application React (Vite)
│   ├── Dockerfile
│   └── .dockerignore
├── server/              # API Express
│   ├── Dockerfile
│   └── .dockerignore
├── screenshots/         # captures d'écran du TP
├── docker-compose.yml   # orchestration des 3 services
├── Jenkinsfile          # pipeline d'automatisation
└── README.md            # ce fichier (explication des étapes)

## 3. Clonage du projet
J'ai d'abord créé mon dépôt GitHub vide, puis je l'ai cloné avec Git Bash :
git clone https://github.com/oumarsacko0209-rgb/TP4-Docker.git
J'ai ensuite cloné le projet fourni par  :
git clone https://github.com/wiem-kb/MyAppReact.git
J'ai copié son contenu (dossiers client et server) dans mon dépôt, puis j'ai fait un premier commit et un push.

## 4. Dockerfiles
### Serveur (`server/Dockerfile`)
- `FROM node:lts-alpine` : image de base Node.js légère (Alpine Linux).
- `WORKDIR /app` : définit le répertoire de travail dans le conteneur.
- `COPY package*.json ./` : copie d'abord les fichiers de dépendances pour profiter du cache Docker.
- `RUN npm install` : installe les dépendances.
- `COPY . .` : copie le reste du code du serveur.
- `ENV MONGO_URI=...` : valeur par défaut de l'URL de connexion à MongoDB (`mongodb` est le nom du conteneur MongoDB sur le réseau Docker).
- `EXPOSE 9000` : indique que l'API écoute sur le port 9000.
- `CMD ["npm", "start"]` : démarre l'application.

### Client (`client/Dockerfile`)
- Les cinq premières instructions ont le même rôle que pour le serveur.
- `RUN npm run build` : construit l'application React pour la production.
- `RUN npm install -g serve` : installe `serve`, un serveur HTTP léger pour servir les fichiers statiques.
- `EXPOSE 3000` : le client est accessible sur le port 3000.
- `CMD ["serve", "-s", "dist", "-l", "3000"]` : sert le dossier `dist` (généré par Vite) sur le port 3000.

### Fichiers `.dockerignore`
Présents dans `client/` et `server/`, ils empêchent de copier `node_modules`, `dist`, `build` et `.git` dans l'image, ce qui accélère le build et allège l'image.

## 5. Construction des images
docker build -t mern-server ./server
docker build -t mern-client ./client
L'option `-t` donne un nom à l'image et le dernier argument indique le dossier contenant le Dockerfile. Dans le sujet, la commande du client contenait une coquille (`-t mern-server`) : j'ai utilisé `mern-client`.

## 6. Lancement manuel
Création du réseau pour que les conteneurs communiquent entre eux :
docker network create mern-network

Lancement de MongoDB :
docker run -d --name mongodb --network mern-network mongo
Lancement du serveur (avec la variable `MONGO_URI`) :
docker run -d --name server --network mern-network -p 9000:9000 -e MONGO_URI=mongodb://mongodb:27017/mern mern-server
Lancement du client :
Vérification avec `docker ps` et `docker logs server`. Le client est accessible sur `http://localhost:3000` et l'API sur `http://localhost:9000`.

![docker ps](screenshots/docker-ps.png)
![application](screenshots/app-manuel.png)

## 7. Docker Compose
Le fichier `docker-compose.yml` à la racine remplace toutes les commandes manuelles précédentes.

- **mongodb** : utilise l'image officielle `mongo`, conteneur nommé `mongodb`, connecté au réseau `mern-network`.
- **server** : construit à partir de `./server`, conteneur nommé `server`, port 9000 exposé, variable d'environnement `MONGO_URI=mongodb://mongodb:27017/mern`, connecté au réseau.
- **client** : construit à partir de `./client`, conteneur nommé `client`, port 3000 exposé, connecté au réseau.
- **depends_on** : définit l'ordre de démarrage (`mongodb` puis `server` puis `client`). Cela garantit l'ordre de lancement, pas que le service soit déjà prêt.
- **networks** : le réseau `mern-network` est de type `bridge`. Les services communiquent grâce au nom du service comme hostname (le serveur joint MongoDB via `mongodb`).
- **MONGO_URI** : variable d'environnement qui permet au serveur Express de savoir où se trouve la base de données.
docker compose up --build
Pour arrêter et supprimer les conteneurs : `docker compose down`.

![compose ps](screenshots/compose-ps.png)
![application](screenshots/app-compose.png)

## 8. Jenkins 
Pour automatiser le déploiement, j'ai créé un `Jenkinsfile` à la racine du projet. Ce pipeline récupère le code depuis GitHub (Checkout), construit les images (`docker compose build`), redéploie les conteneurs (`docker compose down` puis `up -d`) et vérifie leur état (`docker compose ps`).
## 9. Problèmes rencontrés et solutions
- **Conflit de noms de conteneurs** : Docker refusait de créer `mongodb`, `server` et `client` car des conteneurs de même nom existaient déjà. Solution : `docker rm -f client server mongodb`.
- **Erreur 404 sur le client** : `serve` ne trouvait pas de fichiers, car Vite génère le dossier `dist` et non `build`. Solution : remplacer `build` par `dist` dans le `CMD` du Dockerfile.

## 10. Conclusion
Ce TP m'a permis de containeriser une application MERN avec Docker, d'orchestrer ses trois services avec Docker Compose,
 puis d'automatiser le déploiement avec un pipeline Jenkins.