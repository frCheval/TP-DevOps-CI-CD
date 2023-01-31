# TP2

## 1 Qu'est-ce que les testcontainers?

Ce sont des librairies java qui permettent de lancer un ensemble de containers docker pendant les tests. Ici nous utilisons le container postgresql pour l'attacher à notre application pendant les tests.

## 2 Workflow

Création d'un dossier .github/workflows avec un fichier main.yml de base:

```yaml
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: #TODO 
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        #TODO

     #finally build your app with the latest command
      - name: Build and test with Maven
        run: #TODO
```    
      
On voit que dans le fichier main.yml, on a un job test-backend qui va être lancé sur ubuntu-22.04. On a ensuite 3 étapes: checkout, setup-java et build and test with maven. On va donc devoir remplir ces étapes.   

Une fois push, on voit que dans Actions sur GitHub on a bien un job avec le nom du commit qui s'est lancé dans le workflow du name `name: CI devops 2023`. On peut voir que le job a échoué car on n'a pas encore correctement rempli le main.yml.
On rempli les lignes on: push: branches: -main -develop pour que le job se lance sur les branches main et develop lors d'un push.

```yaml
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      - main
      - develop
  pull_request:

```

## 3 CI

Pour l'instant on a un job qui se lance mais qui échoue. On va donc remplir les étapes du job.
L'erreur vient du fait qu'on n'a pas encore installé jdk 17. On va donc utiliser l'action setup-java@v3 pour installer jdk 17, en utilisant la documentation https://github.com/actions/setup-java

Va donc ajouter des steps:

```yaml
- name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'corretto'
          java-version: '17'
```

On a ensuite l'étape build and test with maven. On va donc utiliser la commande mvn clean install pour compiler et tester notre application.

```yaml
- name: Build and test with Maven
        run: mvn clean install
```

On comprend donc les steps s'executent dans l'ordre de haut en bas. On a donc checkout, setup-java et build and test with maven.

On voit que le job est passé avec succès sur GitHub avec le main.yml suivant:
    
```yaml
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      - main
      - develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'corretto'
          java-version: '17'

     #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean install
```

## 4 CD

Les pipelines github Actions nous permettent d'ajouter des étapes de push de nos images sur Docker Hub.

On ajoute donc un nouveau job en dessus de notre job de test qu'on définit comme cela :
``` yaml
build-and-push-docker-image: 
```

Puisque l'on veut une image fonctionnelle, il est important de préciser 
``` yaml
needs: test-backend 
```
juste après la définition de notre jobs. 
Cela nous permet d'indiquer que le job *build-and-push-docker* peut se lancer uniquement si le job *test-backend* est passé.

On ajoute ensuite la ligne 
```yaml
   runs-on: ubuntu-22.04
```
pour préciser la version de ubuntu de notre runner sur laquelle la pipeline va tourner

On définit ensuite 5 step à notre job :
- Checkout code
- Login to DockerHub
- Build image and push backend
- Build image and push database
- Build image and push httpd

### 4.1 Checkout code

L'etape checkout code est definit de cette manière:
```yaml
    - name: Checkout code
    uses: actions/checkout@v2.5.0
```
Cette étape permet de déplacer le code de notre repository sur le runner de notre pipeline.

### 4.2 Login to Docker hub

Cette étape permet de s'authentifier sur la plateforme Docker Hub en prévision d'un push de nos images sur la plateforme.

Elle est définit de cette manière :
```yaml
    - name: Login to DockerHub
    run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}
```

Dans cette étape, on execute la commande docker login. Les credentials utilisés sont des variables secrètes du repository, enregistré au préalable. 
La variable **DOCKERHUB_USERNAME** correspond au nom d'utilisateur de l'utilisateur qui va stocker les images sur Docker Hub.
La variable **DOCKERHUB_TOKEN** correspond à un token généré par Docker Hub pour l'utilisateur. Dans notre cas, le token utilisé est un token généré pour permmettre uniquement la lecture et l'écriture sur les images.

### 4.3 Build and push

Les trois dernières étapes sont le build et push de notre image. Puisque ces étapes sont similaires, j'expliquerai le principe de ces étapes en prenant en exemple la première étape de ce type.

Elle se décrit de cette manière :
```yaml
      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP2/backend
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1-compose-backend-tp:1.0
          push: ${{ github.ref == 'refs/heads/main' }}

```

Chaque étape utilise l'action docker/build-push pour etre effectuer.
L'attribut ```with``` permet d'ajouter des variables à notre étape

On inclue le Dockerfile qui décrit l'initialisation de l'image grace à la variable ```context```.
La variable ```tags``` définit le tag de l'image à-partir d'un username DockerHub et du nom de l'image que l'on souhaite push.

L'attribut ```push``` permet de définir une condition de push de l'image sur Docker Hub : elle n'est push que le commit a lieu sur la branche main.

## 5 Quality Gate

En premier lieu, on va créer une organisation sur sonarcloud. On va donc aller sur https://sonarcloud.io/organizations/create et créer une organisation. On va ensuite garder de côté le `project key` et l'`organization key`.

En deuxième temps, on va créer un token pour sonarcloud. On va donc aller sur https://sonarcloud.io/account/security/ et créer un token. On va ensuite aller dans les settings de notre repo sur GitHub et créer une variable d'environnement secret `SONAR_TOKEN` avec la valeur du token créé précédemment.

Pour lancer le job de qualité, on va utiliser le plugin sonarqube. On va donc modifer l'etape `Build and test with Maven` dans le main.yml:

```yaml
     #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=frCheval_TP-DevOps-CI-CD -Dsonar.organization=devops-2023-cheval-gauchoux -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./TP2/backend/simple-api-student-main
```
On remarque qu'on a set le `project key` à `-Dsonar.projectKey=` et le l'`organization key` à `-Dsonar.organization=`. On aurait put les rentrer en tant que variable d'environnment secret mais ce n'est pas forcément necessaire.
    
On trouve bien le projet sur sonarcloud: https://sonarcloud.io/dashboard?id=frCheval_TP-DevOps-CI-CD. On voit qu'il y a 53.6% de coverage, 0 bugs, 2 code smells et 2 vulnerabilities.     
    
En général dans un projet réel le coverage doit être au dessus de 80%, le projet ne serait donc pas accepté. 
