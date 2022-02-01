# Catherine Vanden Hende - DEVOPS - TP
## 1-1 DATABASE

Variables d'environnements :
>POSTGRES_USER=usr & POSTGRES_PASSWORD=pwd

Construire un reseau : 
>docker network create app-network

Supprimer un reseau : 
>docker network rm app-network

Construire une image : 
>docker build -t dbPostgres .

Construire un container : 
>docker run -d -p 5432:5432 -v /tmp/data:/var/lib/postgresql/data [-e varENV] --network=app-network --name postgres dbPostgres

Supprimer un container démaré : 
>docker rm -f postgres

Adminer SQL : 
>docker run -d -p 8081:8080--network=app-network adminer

Dockerfile :
```docker
FROM postgres:11.6-alpine
COPY 01-CreateScheme.sql  /docker-entrypoint-initdb.d
COPY 02-InsertData.sql  /docker-entrypoint-initdb.d
ENV POSTGRES_DB=db
```

### I.1
>-e sert à donner en lignes de commandes les variables d'environnements (user, password...) lors du run d'un container

### I.2
>Un volume doit être rattaché à la base de données afin de sauvegarder ces données, même lorsque le container est détruit

## 1-2 BACKEND API

>Nous avons besoin d'un build en plusieurs étapes car on doit d'abord builder le package maven avec des jdk (lourds) puis de l'executer avec des jre (plus légers).

Dockerfile :
```dockerfile
# Build
#recupère la version 3.6.3-jdk-11 de maven
FROM maven:3.6.3-jdk-11 AS myapp-build 
#récupere et renvoit le répertoire de mon projet
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
#Copie les fichiers nécessaires pour le build dans le container
COPY simple-api/pom.xml . 
COPY simple-api/src ./src
#package le projet avec maven (sans les tests)
RUN mvn package -DskipTests

# Run
#recupère la version 11 de l'openjdk
FROM openjdk:11-jre
#récupere et renvoit le répertoire de mon projet
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
#Copie le package formé dans le build et l'envoie au container
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
#Run le package et lance le backend
ENTRYPOINT java -jar myapp.jar
```

### Attention a l'environnement 
>mvn dependency:go-offline 

## 1-3 DOCKER-COMPOSE COMMANDS

Build : 
>docker-compose build

Run :
>docker-compose up

## 1-4 DOCKER-COMPOSE FILE

```yml
version: '3.3'

services:
  backend:
    build: ./backend
    networks:
    - my-network
    depends_on: 
    - database

  database:
    build: ./database
    networks:
    - my-network

  httpd:
    build: ./http
    ports: 
    - "80:80"
    networks:
    - my-network
    depends_on: 
    - backend


networks:
  my-network:
```

## 1-5 PUBLISH

Créer un tag pour une image : 
>docker tag devops_database cathvdh/devops_database:1.0

Push une image sur Docker Hub :
>docker push cathvdh/devops_database

Logs/Restart/Stop... :
>docker-compose cmd service

Les images que j'ai mises en ligne sont la base de données, l'api java et le serveur http.

## 2-1 TESTCONTAINERS
>Les testcontainers sont des libraries qui permettent de lancer plusieurs containers en les testant.

## 2-2 GITHUB ACTIONS

```yml
name: CI devops 2022 CPE
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: [master]
  pull_request:
jobs:
  test-backend:
    runs-on: ubuntu-18.04
    steps:
      #checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3
      #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          distribution: 'zulu'
          java-version: '11'
      #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean verify --file ./backend/simple-api/pom.xml
```