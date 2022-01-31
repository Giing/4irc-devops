
# CI/CD - DevOps

## TP 1

### Database
Dockerfile 
```
FROM postgres:11.6-alpine

ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
```
Create network
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
