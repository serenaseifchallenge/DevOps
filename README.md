# TP1

### Database (`database/`)

```powershell
cd database
docker network create app-network   # une seule fois
docker build -t mydatabase .
docker run -d --name mydatabase --network app-network `
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd `
  -v ${PWD}/data:/var/lib/postgresql/data `
  mydatabase
docker run -d -p 8090:8080 --net=app-network --name=adminer adminer
```

Adminer : http://localhost:8090 (serveur `mydatabase`, utilisateur `usr`, mot de passe `pwd`, base `db`).

#### Réponses aux questions 

1-1 For which reason is it better to run the container with a flag -e to give the environment variables rather than put them directly in the Dockerfile?

Mettre les variables (mdp) dans le Dockerfile pose un problème de sécurité, elles sont stockées en clair dans l'image et dans ses couches, ainsi que dans le dépôt Git. Toute personne qui récupère l'image ou le code a donc accès aux identifiants. Avec le flag -e, les valeurs ne sont fournies qu'au lancement du conteneur. L'image ne contient aucun secret et devient réutilisable.

1-2 Why do we need a volume to be attached to our postgres container?

Le système de fichiers d'un conteneur est éphémère donc quand le conteneur est supprimé, toutes ses données disparaissent, y compris le contenu de la base. Un volume monte un espace de stockage de l'hôte sur `/var/lib/postgresql/data`, là où PostgreSQL écrit ses données. Elles sont alors stockées en dehors du conteneur et persistent même s'il est supprimé, recréé ou mis à jour.

1-3 Document your database container essentials: commands and Dockerfile.

Structure :

```
database/
├── Dockerfile
└── sql/
    ├── 01-CreateScheme.sql
    └── 02-InsertData.sql
```

- `FROM postgres:17.2-alpine` : image officielle PostgreSQL, en version Alpine.
- `COPY ./sql/ /docker-entrypoint-initdb.d/` : Copie le contenu de votre répertoire dans le répertoire du conteneur
- `01-CreateScheme.sql` crée les tables `departments` et `students`, puis `02-InsertData.sql` insère les données initiales.

Commandes : 

```powershell
cd database
docker network create app-network   # une seule fois
docker build -t mydatabase .
docker run -d --name mydatabase --network app-network `
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd `
  -v ${PWD}/data:/var/lib/postgresql/data `
  mydatabase
docker run -d -p 8090:8080 --net=app-network --name=adminer adminer
```

- --network app-network : permet aux conteneurs de communiquer entre eux par leur nom (`mydatabase`).
- -e : variables d'environnement de la base.
- -v : volume pour persister les données.
- Adminer (http://localhost:8090) permet de visualiser la base. On s'y connecte avec le serveur mydatabase, l'utilisateur usr, le mot de passe pwd et la base db


### Backend : Hello World (`backend/hello/`)

```powershell
cd backend/hello
javac Main.java (si effectué modif dans main)
docker build -t hello-java .
docker run --rm --name hello-java hello-java
```

### Backend : simple API (`backend/simpleapi/`)

La base de données doit être lancée avant.

```powershell
cd backend/simpleapi
docker build -t simpleapi .
docker run -d --name simple-api --network app-network -p 8080:8080 simpleapi
```

Verifier : http://localhost:8080/departments/IRC/students

#### Réponses aux questions 

1-4 Why do we need a multistage build? And explain each step of this dockerfile.

Un multistage build sépare la compilation de l'exécution dans un seul Dockerfile :
- Pour compiler, il faut un JDK et un mvn
- Pour exécuter, un JRE 

Le premier stage compile l'application. Le second ne récupère que le `.jar` produit et le place dans une image. 

Stage 1 : Build

FROM ...jdk-alpine AS myapp-build : image avec le JDK 21 pour compiler, nommée myapp-build.
ENV MYAPP_HOME=/opt/myapp : variable contenant le dossier de l’application.
WORKDIR $MYAPP_HOME : se place dans ce dossier.
RUN apk add --no-cache maven : installe Maven.
COPY pom.xml . : copie la liste des dépendances.
RUN mvn dependency:go-offline : télécharge les dépendances, mises en cache par Docker.
COPY src ./src : copie le code source.
RUN mvn package -DskipTests : compile et crée le .jar, sans lancer les tests.

Stage 2 : Run

FROM ...jre-alpine : nouvelle image avec seulement le JRE 21.
ENV + WORKDIR : même dossier de travail.
COPY --from=myapp-build ... : récupère uniquement le .jar du stage 1.
ENTRYPOINT ["java", "-jar", "myapp.jar"] : lance l’application au démarrage du conteneur.

### HTTP server (`http-server/`)

Le backend `simple-api` doit être lancé avant.

```powershell
cd http-server
docker build -t my-httpd .
docker rm -f my-httpd
docker run -d --name my-httpd --network app-network -p 80:80 my-httpd
```

Verifier : http://localhost/departments/IRC/students (passe par le reverse proxy)

#### Réponses aux questions

1-5 Why do we need a reverse proxy?

Un reverse proxy est un serveur intermédiaire placé devant nos serveurs applicatifs : il reçoit les requêtes des clients, les transmet au bon backend, puis renvoie la réponse. Sans reverse proxy, chaque serveur est exposé directement sur Internet. Avec un reverse proxy, on aura un point d'entrée unique où on appliquez toutes les règles, une seule fois.

### Link application : Docker Compose (racine du projet)

Arrêter d'abord les conteneurs manuels (`docker rm -f my-httpd simple-api mydatabase adminer`) pour libérer les ports.

```powershell
docker compose up -d --build   # build + lancement des 3 services
docker compose down -v         # arrêt + suppression du volume de la base
```

Verifier : http://localhost/departments/IRC/students

#### Réponses aux questions

1-6 Why is docker-compose so important?

Sans docker-compose, il faut lancer chaque conteneur à la main avec de longues commandes docker run, dans le bon ordre, en créant soi-même les réseaux et les volumes. C'est long, source d'erreurs et difficile à reproduire.

Docker Compose décrit toute l'application dans un seul fichier versionné

On peut alors :
- démarrer, arrêter ou reconstruire toute la stack en une commande 
- obtenir un environnement identique et reproductible sur n'importe quelle machine ;

1-7 Document docker-compose most important commands

```powershell
docker compose up -d --build   # build + lancement des 3 services
docker compose ps              # état des services
docker compose logs -f backend # logs d'un service
docker compose down            # arrêt (garder les données)
docker compose down -v         # arrêt + suppression du volume de la base
```

1-8 Document your docker-compose file

- 3 services, chacun construit depuis son dossier avec build : database (PostgreSQL), backend (API Spring Boot) et httpd (reverse proxy Apache).
- Variables d'environnement : les identifiants viennent du fichier .env et ne sont pas écrits en dur.
- Ségrégation des réseaux : app-network relie httpd et backend, db-network relie backend et database. Le proxy n'a donc aucun accès direct à la base.
- Ports : seul httpd expose un port à l'hôte (80:80). Le backend et la base ne sont accessibles qu'à l'intérieur des réseaux Docker.
- Volume nommé db-data : les données de la base persistent même après un `docker compose down`.
- `healthcheck` + `depends_on`** : la base est testée avec `pg_isready`, et le backend ne démarre que lorsqu'elle est réellement prête (`service_healthy`). `httpd` démarre après le backend.
- `restart: unless-stopped` : les conteneurs redémarrent automatiquement en cas de crash, sauf s'ils ont été arrêtés volontairement.

### Publish (Docker Hub)

```powershell
docker login -u serenaseifchallenge
docker tag devops-database serenaseifchallenge/tp-devops-database:1.0
docker tag devops-backend  serenaseifchallenge/tp-devops-backend:1.0
docker tag devops-httpd    serenaseifchallenge/tp-devops-httpd:1.0
docker push serenaseifchallenge/tp-devops-database:1.0
docker push serenaseifchallenge/tp-devops-backend:1.0
docker push serenaseifchallenge/tp-devops-httpd:1.0
```

#### Réponses aux questions

1-9 Document your publication commands and published images in dockerhub.

```powershell
# Connexion à Docker Hub (avec un access token Read & Write)
docker login -u serenaseifchallenge

# Tag des images avec le nom du compte et une version
docker tag devops-database serenaseifchallenge/tp-devops-database:1.0
docker tag devops-backend  serenaseifchallenge/tp-devops-backend:1.0
docker tag devops-httpd    serenaseifchallenge/tp-devops-httpd:1.0

# Publication
docker push serenaseifchallenge/tp-devops-database:1.0
docker push serenaseifchallenge/tp-devops-backend:1.0
docker push serenaseifchallenge/tp-devops-httpd:1.0
```

- `docker tag` donne à l'image un nom au format `UTILISATEUR/IMAGE:VERSION`. Le préfixe indique le compte Docker Hub, et la version (`1.0`) est plus explicite que `latest`.
- `docker push` envoie l'image sur Docker Hub. Le dépôt est créé automatiquement.

Images publiées :
- `serenaseifchallenge/tp-devops-database:1.0` 
- `serenaseifchallenge/tp-devops-backend:1.0`
- `serenaseifchallenge/tp-devops-httpd:1.0`

Elles peuvent être récupérées sur n'importe quelle machine avec `docker pull serenaseifchallenge/tp-devops-backend:1.0`.

1-10 Why do we put our images into an online repo?

- Partage : les autres membres de l'équipe récupèrent les images avec un simple `docker pull`, sans avoir besoin du code source ni de reconstruire.
- Déploiement : un serveur ou un pipeline CI/CD peut télécharger et lancer directement l'image. C'est exactement la même partout, ce qui évite le « ça marche sur ma machine ».
- Versioning : grâce aux tags, on garde un historique des versions et on peut revenir à une version précédente en cas de problème.
- Sauvegarde : les images ne dépendent plus d'une seule machine locale.

# TP2

2-1 What are testcontainers?

Testcontainers sont des bibliothèques Java qui permettent d'exécuter plusieurs conteneurs Docker pendant les tests

2-2 For what purpose do we need to use secured variables?

Le pipeline a besoin de nos identifiants Docker Hub pour publier les images. Les écrire directement dans le main.yml les rendrait visibles par toute personne ayant accès au dépôt, et ils resteraient dans l'historique Git même après suppression. Sur un dépôt public, n'importe qui pourrait alors publier des images à notre place. Les secrets GitHub résolvent ce problème, ils sont chiffrés et stockés par GitHub

2-3 Why did we put `needs: build-and-test-backend` on this job? 

Dans notre workflow, le job de tests s'appelle `test-backend`, donc la ligne est `needs: test-backend`.

Par défaut, GitHub Actions lance tous les jobs d'un workflow en parallèle. Sans needs, le job build-and-push-docker-image démarrerait en même temps que test-backend, sans attendre son résultat. Les images Docker seraient alors construites et publiées même si la compilation ou les tests échouent.

2-4 For what purpose do we need to push docker images?

Les publier sur Docker Hub permet de :
- Continuous Delivery : chaque commit validé sur `main` produit une nouvelle version de l'application, prête à être déployée
- Déployer partout : N'importe qui récupère l'image avec docker pull, sans avoir le code source ni devoir rebuild
- Garantir que ce qui est déployé a été testé : grâce à needs: test-backend, seules les versions dont les tests passent sont publiées
- Garder un historique des versions, et pouvoir revenir à une image précédente en cas de problème

Bonus TP2 : 

Proteger la branche main

# TP3 

3-1 Document your inventory and base commands 

fichier yaml :

- `all` : le groupe racine, qui contient tous les serveurs.
- `vars` : variables communes à tous les serveurs.
  - `ansible_user` : l'utilisateur SSH (`admin`, utilisateur par défaut sur Debian).
  - `ansible_ssh_private_key_file` : le chemin de la clé privée. On n'a plus besoin de passer `-u` et `--private-key` à chaque commande.
- `children` → `prod` : un groupe de serveurs. On peut créer plusieurs groupes (`prod`, `staging`, `database`…) pour cibler des machines selon leur rôle.
- `hosts` : la liste des serveurs du groupe

commandes : 

ansible all -i inventories/setup.yml -m ping : Vérifie que le serveur est joignable et qu'Ansible peut s'y authentifier (réponse `pong`) 

ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*" : Récupère les facts liés à la distribution du serveur (Les facts sont des informations découvertes automatiquement par Ansible sur le serveur (système, version, IP, mémoire…), préfixées par ansible_. Le module `setup` permet de les afficher.)

ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become : Désinstalle Apache


3-2 Document your playbook

```
ansible/
├── inventories/setup.yml     # inventaire (serveur, utilisateur, clé)
├── playbook.yml              # playbook principal : appelle les rôles
└── roles/
    └── docker/
        ├── tasks/main.yml    # tâches d'installation de Docker
        └── handlers/main.yml 
```

Playbook :

- hosts: all : s'applique à tous les serveurs de l'inventaire.
- gather_facts: true` : récupère les facts du serveur. 
- become: true : toutes les tâches s'exécutent en root, ce qui est nécessaire pour installer des paquets.
- roles : la liste des rôles à appliquer, dans l'ordre. Le playbook reste court et lisible : chaque rôle contient la logique d'une partie de l'infrastructure.

Install required packages : installe les prérequis 
Create keyrings directory : crée `/etc/apt/keyrings` 
Add Docker GPG key : télécharge la clé qui authentifie les paquets Docker 
Add Docker APT repository :  ajoute le dépôt Docker correspondant à la version de Debian (fact `distribution_release`) 
Install Docker : installe `docker-ce` 
Install Python3 and pip3 : installe Python et pip 
Install Docker SDK for Python : installe le SDK Docker dans un environnement virtuel `/opt/docker_venv`, nécessaire aux modules `docker_container` et `docker_network` 
Make sure Docker is running : s'assure que le service Docker est démarré 


3-3 Document your docker_container tasks configuration.

- Chaque partie de l’application a son propre rôle Ansible : network, database, app et proxy.
- Les conteneurs sont lancés avec le module community.docker.docker_container, à partir des images publiées sur Docker Hub par GitHub Actions
- Les variables d'environnement sont centralisées dans var_env/all.yml, chargé dans le playbook avec vars_files.
- Le réseau app-network est créé avec le module docker_network. Les conteneurs s’y retrouvent par leur nom.
- database : conteneur PostgreSQL, configuré par variables d’environnement (POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD), avec un volume db-data pour garder les données.
- app : conteneur de l’API Spring Boot, qui reçoit les variables de connexion à la base lues par application.yml.
- proxy : conteneur Apache, qui reçoit l’adresse du backend. C’est le seul conteneur exposé, sur le port 80.

Le playbook est séparé en deux parties : l’installation de Docker utilise le Python du système, et le déploiement utilise le Python du venv /opt/docker_venv, qui contient le SDK Docker (ansible_python_interpreter).

Les rôles s’exécutent dans l’ordre réseau → base → API → proxy, car chacun dépend du précédent.
Seul le proxy est accessible depuis Internet. La base et l’API restent internes au réseau Docker.
