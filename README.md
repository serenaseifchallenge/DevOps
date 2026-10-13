## Comment lancer chaque partie

> Commandes PowerShell, à lancer depuis le dossier indiqué.
> Pour relancer un conteneur existant, le supprimer d'abord avec `docker rm -f <nom>`.

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

### Backend : Hello World (`backend/hello/`)

```powershell
cd backend/hello
docker build -t hello-java .
docker run --rm --name hello-java hello-java
```

Le Dockerfile est multistage : la compilation de `Main.java` se fait dans Docker (JDK 21), donc pas besoin de `javac` en local.

### Backend : simple API (`backend/simpleapi/`)

La base de données doit être lancée avant.

```powershell
cd backend/simpleapi
docker build -t simpleapi .
docker run -d --name simple-api --network app-network -p 8080:8080 simpleapi
```

Test : http://localhost:8080/departments/IRC/students

### HTTP server (`http-server/`)

Le backend `simple-api` doit être lancé avant.

```powershell
cd http-server
docker build -t my-httpd .
docker rm -f my-httpd
docker run -d --name my-httpd --network app-network -p 80:80 my-httpd
```

Test : http://localhost/departments/IRC/students (passe par le reverse proxy)

### Link application : Docker Compose (racine du projet)

Remplace tous les lancements manuels ci-dessus : arrêter d'abord les conteneurs manuels
(`docker rm -f my-httpd simple-api mydatabase adminer`) pour libérer les ports.

```powershell
docker compose up -d --build   # build + lancement des 3 services
docker compose ps              # état des services
docker compose logs -f backend # logs d'un service
docker compose down            # arrêt (garder les données)
docker compose down -v         # arrêt + suppression du volume de la base
```

Test : http://localhost/departments/IRC/students

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