# TP1

## Database 

On veut build une image postgresql:14.1 alpine

On reprend le fichier Dockerfile de l'énoncé qui définit la BDD 'db' et l'utilisateur 'user'avec le mot de passe 'pwd'.

On execute ensuite la commande suivante pour build

``` docker build -t fcheval/database-tp1 .```

On crée ensuite un network qui nous servira juste après :

```docker network create app-network```

On lance ensuite notre appplication de BDD avec la commande suivante :

```docker run -d --net=app-network --name database-tp1 fcheval/database-tp1```

On utilise la commande -d pour le run en background.

On lance ensuite un conteneur adminer pour afficher notre base de données :

```
docker run \
-p "8090:8080" \
--net=app-network \
--name=adminer \
-d \
adminer
```

On peut accéder à l'interface de la connexion BDD via localhost:8090 et se connecter à la BDD avec les crédentials :
serveur : database-tp1:5432
user: usr
password : pwd
databse: db

Pour ajouter des données au démarrage on copie des script sql dans le dossier /docker-entrypoint-initdb.d de notre conteneur. Pour cela, on ajoute les lignes suivantes au DockerFile :

```COPY CreateSchema.sql /docker-entrypoint-initdb.d/
COPY InsertData.sql /docker-entrypoint-initdb.d/```

On souhaite persister les données en cas de destruction de notre conteneur. Pour cela on complète notre commande docker run avec l'attribut suivant :
```-v volume:/var/lib/postgresql/data```

Cela bin notre dossier volume en local au dossier /var/lib/postgresql/data de notre conteneur

## Backend-api

Après inspection de la plateforme openjdk sur dockerhub, on choisit de copier le jre pour java 11 avec la commannde suivant :

```docker pull openjdk:11-jre```

## http-server