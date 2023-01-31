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
COPY InsertData.sql /docker-entrypoint-initdb.d/ 
```

On souhaite persister les données en cas de destruction de notre conteneur. Pour cela on complète notre commande docker run avec l'attribut suivant :

```-v volume:/var/lib/postgresql/data```

Cela bin notre dossier volume en local au dossier /var/lib/postgresql/data de notre conteneur

## Backend-api

Après inspection de la plateforme openjdk sur dockerhub, on choisit de copier le jre pour java 11 avec la commannde suivant :

```docker pull openjdk:11-jre```

On constitue notre fichier Dockerfile, après avoir généré le fichier. class à l'aide de la commande 
```javac Main.java```

On copie sur dans le conteneur le fichier class et on run notre application.

Au final on a un Dockerfile de ce site :

```

#image de base
FROM openjdk:11-jre

# copy files required for the app to run
COPY Main.class /

#Compiled class
RUN java Main
```

On build notre image avec la commande suivante :

```docker build -t fcheval/backend-api-tp1 .```

et ensuite :

```docker run -d --net=app-network --name backend-api-tp1 fcheval/backend-api-tp1```

La suite du TP n'est qu'une suite d'instruction pour créer un build multistage.

### Question 1-2 : Pourquoi avons-nous besoin d'un build multistage ? Explication du fichier Dockerfile :

``` 
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
```
La première ligne du fichier nous indique nous allons utiliser une image maven pour compiler notre code java

```
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
```
Cette étape définit la variable d'environnement MYAPP_HOME à /opt/myapp.
La commande WORKDIR définit le dossier ou les commandes suivantes seront executé.

```
COPY pom.xml .
COPY src ./src
```
Ces commandes copient le fichier pom.xml dans le répertoire /opt/myapp du conteneur et copier le dossier src local dans le répertoire /opt/myapp

```
RUN mvn package -DskipTests
```

Cette commande crée le fichier jar à l'aide maven.

```
FROM amazoncorretto:17
```
Pour cette étape, on indique qu'on utilise cette image qui correspond au jre en java 17.

```
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
```
Cette étape définit la variable d'environnement MYAPP_HOME à /opt/myapp.
La commande WORKDIR définit le dossier ou les commandes suivantes seront executé.

```
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```
On copie le point jar au point endroit et on le définit en tant que point d'entrée à executer.

## http-server

On peut se fier à la documentation sur docker hub pour l'image httpd :
https://hub.docker.com/_/httpd

On construit le Dockerfile suivant :
```
FROM httpd:2.4
COPY ./public-html/ /usr/local/apache2/htdocs/
```

On met le fichier html dans le dossier public-html et on build notre image avec la commande suivante :
``` docker build -t fcheval/http-serve-tp1 .```

et on lance notre conteneur avec la commande 
```docker run -d --net=app-network -p 80:80 --name http-serve-tp1 fcheval/http-serve-tp1```

Si on veut afficher la conf httpd, on peut utiliser la commande 
```docker exec -it http-serve-tp1 cat /usr/local/apache2/conf/httpd.conf ```

ou on peut copier la fichier pour le modifier en local avec la commande suivante 
``` docker cp http-serve-tp1:/usr/local/apache2/conf/httpd.conf . ```

Pour la suite du tp, on décide de  simplifier la gestion de nos conteneurs en utilisant docker-compose. 

La suite se trouve [ici](../TP1-Compose/)