# TP1

## 1.3 - Docker Compose Commandes

Les commandes les plus importantes sont:
* `docker-compose build`, qui permet de build les containers.
* `docker-compose up -d`, qui permet de lancer les containers en arrière plan.
* `docker-compose up -d --build`, qui permet de build les containers qui ont été modifiés et de les lancer en arrière plan.
* `docker-compose down`, qui permet de stopper les containers.
* `docker-compose ps`, qui permet de voir les containers en cours d'exécution.
* `docker-compose logs <container>`, qui permet de voir les logs des containers.
* `docker-compose exec <container> <commande>`, qui permet d'exécuter une commande dans un container.
* `docker-compose exec <container> bash`, qui permet d'entrer dans un container en mode interactif (bash mais peut être autre chose).
* `docker-compose images`, qui permet de voir les images docker.
* `docker-compose rm`, qui permet de supprimer les containers.

## 1.4 - Docker Compose File

Le fichier `docker-compose.yml` permet de définir les containers et les images docker à utiliser. Il est possible de définir des variables d'environnement, des volumes, des ports, des dépendances, etc.
On lui donne une version qui doit correspondre avec celle de sa version de docker compose.

Ensuite une liste de `service` est créée. Chaque service est défini par un nom:
* `image`, qui permet de définir l'image docker à utiliser (ex postgres:latest).
* `build`, qui permet de définir le chemin du Dockerfile à utiliser.
* `container_name`, qui permet de définir le nom du container.
* `environment`, qui permet de définir les variables d'environnement.
* `volumes`, qui permet de définir les volumes, les fichiers qu'on veut partager entre le container et l'hôte.
* `ports`, qui permet de définir les ports (ex port hote 8080:80 port container).
* `depends_on`, qui permet de définir les dépendances entre les containers.
* `command`, qui permet de définir la commande à exécuter au lancement du container.

## 1.5 - Docker Publish Image

Avant de pouvoir push une image il faut la tagger. Pour cela on trouve une image docker avec `docker images` et ensuite on tag l'image avec `docker tag <image> <user>/<image>:<tag>`. On peut ensuite push l'image avec `docker push <user>/<image>:<tag>`.

```
docker images
docker tag tp1-compose_postgres-tp xgauchoux/tp1-compose_postgres-tp:1.0
docker push xgauchoux/tp1-compose_postgres-tp:1.0
```
    
Resultat: https://hub.docker.com/repositories/xgauchoux




