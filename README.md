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

## 2-3 QUALITY GATE CONFIGURATIONS
```yml
name: CI devops 2022 CPE
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: [master]
  pull_request:
env:
  GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}

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
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=VandenHendeCatherine_DEVOPS -Dsonar.organization=vandenhendecatherine -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{secrets.SONAR_TOKEN }} --file ./backend/simple-api/pom.xml

# define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest
    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

        # Login to docker with github secrets
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_TOKEN }}

      - name: Build image and push backend
        uses: docker/build-push-action@v2
        with:
      # relative path to the place where source code with Dockerfile is located
          context: ./backend
      # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/backend
          push: ${{ github.ref == 'refs/heads/master' }}

      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
      # relative path to the place where source code with Dockerfile is located
          context: ./database
      # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/database
          push: ${{ github.ref == 'refs/heads/master' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
      # relative path to the place where source code with Dockerfile is located
          context: ./http
      # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/httpd
          push: ${{ github.ref == 'refs/heads/master' }}    
```

## 3-1 ANSIBLE COMMANDS

inventory.yml :
```yml
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: Ansible/id_rsa
  children:
    prod:
    hosts: centos@catherine.vanden-hende.takima.cloud
```
Version Ansible :
>ansible --version

Connection ssh :
>ssh -i Ansible/id_rsa centos@catherine.vanden-hende.takima.cloud

Ping :
>ansible all -m ping -i inventory.yml

Installation Apache :
>ansible all -m yum -a “name=httpd state=present” -i inventory.yml --become

Création page html :
>ansible all -m shell -a 'echo `<html><h1>Hello CPE</h1></html>` >>
/var/www/html/index.html' -i inventory.yml
--become

Démarrer Apache :
>ansible all -m service -a "name=httpd state=started"
-i inventory.yml --become

Afficher la distribution OS : 
>$ ansible all -i inventory.yml -m setup -a
"filter=ansible_distribution*"

Desinstaller Apache :
>ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent"
--become

## 2-3 PLAYBOOK

Initialisaion des roles :
>ansible-galaxy init roles/docker

Lancer un playbook :
>ansible-playbook -i inventory.yml playbook.yml

Playbook :
```yml
- hosts: all
  gather_facts: false
  become: true

  roles: 
  - docker
```

Docker main.yml :
```yml
 # Install Docker
    - name: Clean packages
      command:
        cmd: dnf clean -y packages
    - name: Install device-mapper-persistent-data
      dnf:
        name: device-mapper-persistent-data
        state: latest
    - name: Install lvm2
      dnf:
        name: lvm2
        state: latest
    - name: add repo docker
      command:
        cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
    - name: Install Docker
      dnf:
        name: docker-ce
        state: present
    - name: install python3
      dnf:
        name: python3
    - name: Pip install
      pip:
        name: docker
    - name: Make sure Docker is running
      service: name=docker state=started
      tags: docker
```

## 3-3 Deployement
```yml
#playout.yml
- hosts: all
  gather_facts: false
  become: true

  roles: 
  - docker
  - network
  - database
  - app
  - proxy

# tasks file for roles/docker : 0 changes
# tasks file for roles/network
- name: Create a network
  docker_network:
    name: my-network

# tasks file for roles/database
- name: Create a data container
  docker_container:
    name: database
    image: cathvdh/database
    volumes:
      - /data
    networks: 
      - name: my-network

# tasks file for roles/app
- name: Create a app container
  docker_container:
    name: backend
    image: cathvdh/backend
    networks: 
      - name: my-network

# tasks file for roles/proxy
- name: Create a proxy container
  docker_container:
    name: httpd
    image: cathvdh/httpd
    ports: 
    - "80:80"
    networks: 
      - name: my-network
```