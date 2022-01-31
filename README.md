# Catherine Vanden Hende - DEVOPS - TP
## Commandes basiques
Construire un reseau : docker network create my-net
Supprimer un reseau : docker rm my-net

Construire une image : docker build -t nameImage .

Construire un container : docker run -d [-p portext:portcont ][-e varENV] [--network=app-network] --name nameApp nameImage
Supprimer un container : docker rm nameApp

Adminer SQL : $ docker run --link some_database:db -d [-p 8080:8080] [--network=app-network] adminer

## DATABASE

-e sert à donner en lignes de commandes les variables d'environnements (user, password...) lors du run d'un container
