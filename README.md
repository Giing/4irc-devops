
# CI/CD - DevOps

# TP 1

## Summary
1.Database
2.Backend API
3.HTTP server
4.Link application

### Database
#### Dockerfile 
```
FROM postgres:11.6-alpine

ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
```
#### Create network
`docker network create app-network`
Run the postgres database
`docker run -p 5432:5432 --name postgres-database --network=app-network database`
Run adminer
`docker run --network=app-network --name adminer -p 8080:8080 adminer`
__Question__ : Why should we run the container with a flag -e to give the environment variables ? 
__Réponse__ : Cela permet plus de souplesse car on peut initialiser nos variables d'environnements nous même

Pour que nos scripts sql s'exécutent au démarrage il faut que nos scripts se trouvent dans le dossier `/docker-entrypoint-initdb.d`. Il faut modifier le dockerfile comme suivant :
```
FROM postgres:11.6-alpine
COPY ./scripts /docker-entrypoint-initdb.d

ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
```
Création d'un volume pour la persistance des données
`docker run -p 5432:5432 --name postgres-database -v /data/:/var/lib/postgresql/data --network=app-network database`
__Question__ : Why do we need a volume to be attached to our postgres container ?
__Réponse__ : Attacher un volume à la base de données permet de conserver les données mêmes quand le container est stoppé. Cela est indispensable pour la persistance des données.

### Backend API
```
FROM openjdk:16-alpine3.13
COPY ./src /usr/src/api
WORKDIR  "/usr/src/api"
CMD ["java", "Main"]
```
`docker build -t api-backend .`

`docker run --name backend-api -it api-backend`

__Question__ : Why do we need a multistage build ? And explain each steps of this dockerfile
__Réponse__ : 
```
# Build
FROM maven:3.6.3-jdk-11 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT java -jar myapp.jar
```
On utilise un multistage build pour compiler notre application java dans un premier container, puis pour la lancer dans un second container. On fait cela pour avoir une image la plus optimisée possible.

### HTTP Server
Pour rapatrier un fichier du container

`docker cp http-server:/usr/local/apache2/conf/httpd.conf .`

#### Reverse proxy

Il faut ajouter dans notre fichier httpd.conf le codes suivant :

```
ServerName localhost

<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://simple-api-main:8080/
    ProxyPassReverse / http://simple-api-main:8080/
</VirtualHost>
```

et décommenter les lignes suivantes :
```
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```
_Attention_ : Il faut que la base de données, l'api et le serveur http soit sur le même réseau. Ne pas oublier `--network=app-network`

__Question__ : Why do we need a reverse proxy ?

__Réponse__ : On utilise un reverse proxy our exposer certaines ressources de notre serveur. On veut qu'un utilisateur extérieur puisse accéder à notre api mais pas à notre base de données.
### Link application
#### Docker compose

__Question__ : Why is docker-compose so important ?

__Réponse__ : Docker compose permet de build plusieurs image et de lancer plusieurs container via une seule commande et un seul fichier de conf. 

__Most important commands__:
- `docker-compose up` : build et crée les images et démarre les containers pour tous les services listés dans le docker-compose.yml
- `docker-compose down` : stop et supprime les containers démarrés

```
version: '3.3'
services:
  simple-api-main:
    build: ./simple-api-main/simple-api # dossier cible pour build l'image (où se trouve le dockerfile)
    networks: # réseau lié au container
      - app-network
    depends_on: # dépends de la base de données pour être lancé
      - postgres-database
  postgres-database:
    build: ./database # dossier cible pour build l'image
    networks: # réseau lié au container
      - app-network 
  httpd: 
    build: ./http_server # dossier cible pour build l'image
    ports: # liste des ports à exposer
      - "80:80"
    networks: # réseau lié au container
      - app-network
    depends_on: # dépends de simple-api-main pour être lancé
      - simple-api-main
networks: # liste des réseaux
  app-network:
```

#### Publish
- database
`docker tag 4ircdevops_postgres-database giing/postgres-database:1.0`
`docker push giing/postgres-database:1.0`
__link__ : https://hub.docker.com/r/giing/postgres-database
- api
`docker tag 4ircdevops_simple-api-main giing/simple-api-main:1.0`
`docker push giing/simple-api-main:1.0`
__link__ : https://hub.docker.com/r/giing/simple-api-main
- http server
`docker tag 4ircdevops_httpd giing/http-server:1.0`
`docker push giing/http-server:1.0`
__link__ : https://hub.docker.com/r/giing/http-server

# TP 2

__Question__ : What are testcontainers?
__Réponse__ : Les testcontainers sont des librairies java qui permettent de lancer des containers docker pendant les tests.