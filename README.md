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
