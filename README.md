# TP1

## Database

```sh
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY ./initdb /docker-entrypoint-initdb.d/
```

- **FROM** définit l'image de base du conteneur, dans ce cas, PostgreSQL.  
- **COPY** transfère un fichier depuis la machine hôte vers un répertoire spécifique du conteneur Docker.  
- **ENV** sert à définir des variables d'environnement au sein du conteneur Docker.

```sh
# Create network, communication entre 2 conteneurs, il faut créer un réseau commun

docker network create app-network
```

```sh
# To restart admirer

docker run -d --name adminer --network app-network -p 8090:8080 adminer
```

```sh
# Build docker image

docker build -t mypostgres .
```

```sh
# Run the PostgreSQL Container with Volume

docker run -p 5432:5432 --net=app-network --name mypostgres -v data:/var/lib/postgresql/data mypostgres

-p exposer les ports du conteneur sur la machine hôte.  
--name sert à attribuer un nom au conteneur.  
-v gère les volumes pour la persistance des données, sinon si container détruit, données conservées
```
```sh
#Run Adminer

docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
```

**1-1 Why should we run the container with a flag -e to give the environment variables?**

Utiliser l'option `-e` pour passer des variables d'environnement lors de l'exécution est plus sûr que de les écrire directement dans le Dockerfile. Cela évite de stocker des informations sensibles, comme des mots de passe, en clair dans le fichier, ce qui pourrait poser un risque si le Dockerfile est partagé ou enregistré dans un gestionnaire de versions.

**1-2 Why do we need a volume to be attached to our postgres container?**

Attacher un volume au conteneur PostgreSQL garantit que les données de la base de données sont stockées de manière persistante sur la machine hôte. Ainsi, même si le conteneur est détruit ou recréé, les données restent intactes et peuvent être réutilisées par la nouvelle instance du conteneur.

**1-3 Document your database container essentials: commands and Dockerfile.**

Done

## Backend API (1-4)

###  Build
```sh
# Build
FROM maven:3.9.9-amazoncorretto-21 AS myapp-build
ENV MYAPP_HOME=/opt/myapp 
WORKDIR $MYAPP_HOME
COPY /simpleapi/pom.xml .
COPY /simpleapi/src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:21
ENV MYAPP_HOME=/opt/myapp 
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT ["java", "-jar", "myapp.jar"]
```
Un build multistage permet d'utiliser plusieurs instructions FROM dans un seul Dockerfile. Chaque instruction FROM démarre une nouvelle étape, et nous pouvons sélectionner les artefacts à copier d'une étape à l'autre.
Le build contient le JDK nécessaire pour compiler l'application avec Maven
Le run contient uniquement le JRE, suffisant pour éxecuter l'application déjà compilée, rendant l'image plus légère.
La première étape génère les artefacts nécessaires, tandis que la seconde étape ne copie que les artefacts essentiels dans l'image finale, l'image finale sera plus petite et performante.

### Explication du Dockerfile

#### 1. **Phase de Build**

```dockerfile
FROM maven:3.9.9-amazoncorretto-21 AS myapp-build
```
Utilise Maven et le JDK Amazon Corretto pour compiler l'application.

```dockerfile
WORKDIR /opt/myapp
COPY /simpleapi/pom.xml .
COPY /simpleapi/src ./src
RUN mvn package -DskipTests
```
Définit le répertoire de travail, copie le code source et compile l'application en un fichier JAR.

#### 2. **Phase d'Exécution**

```dockerfile
FROM amazoncorretto:21
```
Utilise l'image JRE Amazon Corretto pour l'exécution de l'application.

```dockerfile
COPY --from=myapp-build /opt/myapp/target/*.jar /opt/myapp/myapp.jar
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```
Copie le JAR compilé et exécute l'application avec `java -jar`.

---

Cette approche sépare la construction (avec JDK) et l'exécution (avec JRE) pour une image finale plus légère et optimisée.

## Http server

**1-5 Why do we need a reverse proxy?**

Un reverse proxy est utilisé pour rediriger les requêtes des clients vers un ou plusieurs serveurs backend, en agissant comme un intermédiaire. 
Avantages : sécurité en masquant le serveur backend, gestion du SSL/TLS etc ..
Dans notre cas, il redirige les requêtes HTTP vers l'application backend de manière simple.

```sh

# database
docker build -t mypostgres .
docker run -p 5432:5432 --net=app-network --name mypostgres -v data:/var/lib/postgresql/data mypostgres

# API
docker build -t simple-api-student .
docker run -p 8080:8080 --net=app-network --name simple-api-student simple-api-student

# Server, 8040 car port 80:80 ne marchait vraiment pas
docker build -t simple-http-server .
docker run -p 8040:80 --net=app-network --name simple-http-server -d simple-http-server

# To check
docker stats simple-http-server
docker inspect simple-http-server
docker logs simple-http-server

```

`Dockerfile` :
```dockerfile
FROM httpd:2.4

COPY httpd.conf /usr/local/apache2/conf/httpd.conf
COPY ./index.html /usr/local/apache2/htdocs/
```

`httpd.conf` que on a récupéré avec : 
```sh
docker cp simple-http-server:/usr/local/apache2/conf/httpd.conf ./httpd.conf
```
```sh
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://simple-api-student:8080/
ProxyPassReverse / http://simple-api-student:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

Avantages de l'utilisation de `docker cp` pour récupérer `httpd.conf` :

- **Personnalisation** : Ajuster la configuration selon les besoins.
- **Cohérence** : Configuration par défaut qui fonctionne, puis y apporter des modifications.
- **Contrôle de version** : Garder le fichier de configuration personnalisé, avec le Dockerfile et autres fichiers.

## Link application

**1-6 Why is docker-compose so important?**

Docker-compose est important car il permet de gérer plusieurs conteneurs Docker avec une seule commande. Il simplifie le déploiement et l'organisation d'applications utilisant plusieurs conteneurs, en facilitant le démarrage, l'arrêt et la gestion des services d'une application. Cela permet de gagner du temps et d'assurer que la configuration est fiable et peut être reproduite facilement.

---

**1-8 Document docker-compose most important commands.**

- **Démarrer les conteneurs** :  
  ```shell
  docker-compose up
  ```

- **Démarrer les conteneurs en mode détaché** (en arrière-plan) :  
  ```shell
  docker-compose up -d
  ```

- **Arrêter et supprimer les conteneurs** :  
  ```shell
  docker-compose down
  ```

- **Mettre en pause les conteneurs** :  
  ```shell
  docker-compose pause
  ```

- **Reprendre les conteneurs mis en pause** :  
  ```shell
  docker-compose unpause
  ```

- **Voir les logs des conteneurs** :  
  ```shell
  docker-compose logs
  ```

- **Construire les images** :  
  ```shell
  docker-compose build
  ```

---

**1-8 Document your docker-compose file.**

Voici un exemple de structure de fichier `docker-compose.yml` :

```yaml
version: '3.7'

services:
  backend:
    build:
      context: ./backend_api/app2
      dockerfile: Dockerfile
    container_name: simple-api-student
    networks:
      - app-network
    depends_on:
      - database

  database:
    build:
      context: ./postgres
      dockerfile: Dockerfile
    container_name: mypostgres
    networks:
      - app-network
    volumes:
      - ./data:/var/lib/postgresql/data

  httpd:
    build:
      context: ./http_server
      dockerfile: Dockerfile
    container_name: simple-http-server
    networks:
      - app-network
    ports:
      - 8040:80

networks:
  app-network:
```

#### services

1. **backend** : 
   - Construit l'image à partir de `./backend_api/app2` avec le Dockerfile.
   - Dépend de `database` et utilise le réseau `app-network`.

2. **database** : 
   - Construit l'image à partir de `./postgres`.
   - Utilise un volume pour persister les données dans `./data` et se connecte au réseau `app-network`.

3. **httpd** : 
   - Construit l'image à partir de `./http_server`.
   - Expose le port `8040` local vers le port `80` du conteneur et utilise le réseau `app-network`.

---

#### networks

```yaml
networks:
  app-network:
```
Crée un réseau `app-network` pour que tous les services puissent communiquer entre eux.

En résumé, ce fichier configure un backend, une base de données PostgreSQL et un serveur HTTP, tous connectés via un réseau privé.

---

## Publish

**1-9 Document your publication commands and published images in dockerhub.**

```shell
# Se connecter à Docker Hub :
docker login

#Taguer votre image :
docker tag mypostgres wanoni/my-database:1.0

# Pousser l'image vers Docker Hub :
docker push wanoni/my-database:1.0

# de même avec api et server
```

**1-10 Why do we put our images into an online repo?**

Mettre nos images dans un dépôt en ligne comme Docker Hub permet de les partager facilement avec d'autres, de les sauvegarder, d'y accéder à distance et de gérer les versions. C'est essentiel pour collaborer et déployer de manière efficace.

---

# TP2 : Github Actions

Je me suis servi de la correction pour ce tp : https://github.com/emr02/tp-devops-correction-docker

**2-1 What are testcontainers?**

They simply are java libraries that allow you to run a bunch of docker containers while testing. Here we use the postgresql container to attach to our application while testing. 

**2-2 Document your Github Actions configurations.**

Github Action pour tester le backend à chaque commit sur la branch main :

```yml
name: CI devops 2025
on:
  #to begin you want to launch this job in main and develop
  push:
    branches:
      - main
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
      # Checkout your GitHub code using actions/checkout@v4
      - uses: actions/checkout@v4

      # Set up JDK 21 using actions/setup-java@v3
      - name: Set up JDK 21
        uses: actions/setup-java@v3
        with:
          distribution: 'corretto'
          java-version: '21'

      # Build and test the project using Maven
      - name: Build and test with Maven
        # enlever --file ./simple-api/pom. et changer sonar.logn to sonar.token
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=devops-project12345 -Dsonar.organization=devops-sonar123 -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }}
        working-directory: simple-api

  # Define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # Run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04

    # Steps to perform in job
    steps:
      # Checkout the code
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      # Log in to Docker Hub using the provided credentials
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}

      # Build the Docker image for the backend and push it to Docker Hub
      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # Relative path to the place where source code with Dockerfile is located
          context: ./simple-api
          # Note: tags have to be all lower-case
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest
          # Build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      # Build the Docker image for the database and push it to Docker Hub
      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-database:latest
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      # Build the Docker image for the HTTP server and push it to Docker Hub
      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./http-server
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-httpd:latest
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}
```
**2-3 For what purpose do we need to push docker images?**

On pousse des images Docker pour simplifier le déploiement, rendre l’application accessible depuis n’importe quel serveur ou cloud et automatiser les mises à jour sans configuration complexe. 
**2.4**
```yml
mvn -B verify sonar:sonar -Dsonar.projectKey=devops-project12345 -Dsonar.organization=devops-sonar123 -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }}
```
Cette commande exécute **Maven** pour tester et analyser le code avec **SonarCloud** :  

### Décomposition :
1. **`mvn -B`** → Exécute Maven en mode "batch" (évite les interactions utilisateur).
2. **`verify`** → Compile et teste le projet pour vérifier son bon fonctionnement.
3. **`sonar:sonar`** → Lance l'analyse du code avec **SonarCloud** (vérification de la qualité du code, détection de bugs, etc.).
4. **`-Dsonar.projectKey=devops-project12345`** → Identifie le projet dans **SonarCloud**.
5. **`-Dsonar.organization=devops-sonar123`** → Spécifie l'organisation SonarCloud associée.
6. **`-Dsonar.host.url=https://sonarcloud.io`** → Indique l’URL du serveur SonarCloud.
7. **`-Dsonar.token=${{ secrets.SONAR_TOKEN }}`** → Utilise un **jeton secret** pour s’authentifier et envoyer les résultats sur SonarCloud.

### En résumé :
👉 **Cette commande compile, teste et envoie l'analyse du code à SonarCloud pour évaluer sa qualité.** ✅


## Bonus: split pipelines (Optional)

Build
```yml
name: Test Backend

on:
  push:
    branches:
      - main
      - develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
      # Checkout your GitHub code using actions/checkout@v4
      - uses: actions/checkout@v4

      # Set up JDK 21 using actions/setup-java@v3
      - name: Set up JDK 21
        uses: actions/setup-java@v3
        with:
          distribution: 'corretto'
          java-version: '21'

      # Build and test the project using Maven
      - name: Build and test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=devops-project12345 -Dsonar.organization=devops-sonar123 -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }}
        working-directory: simple-api

```
Deploy

```yml
name: Build and Push Docker Image

on:
  workflow_run:
    workflows: ["Test Backend"]
    types:
      - completed
    branches: 
      - main

jobs:
  build-and-push-docker-image:
    if: github.event.workflow_run.conclusion == 'success' &&
      github.event.workflow_run.head_branch == 'main' # Ensure deploy only runs for main
    runs-on: ubuntu-22.04

    steps:
      # Checkout the code
      - name: Checkout code
        uses: actions/checkout@v4

      # Log in to Docker Hub using the provided credentials
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}

      # Build the Docker image for the backend and push it to Docker Hub
      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # Relative path to the place where source code with Dockerfile is located
          context: ./simple-api
          # Note: tags have to be all lower-case
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest
          # Build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      # Build the Docker image for the database and push it to Docker Hub
      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-database:latest
          # Build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      # Build the Docker image for the HTTP server and push it to Docker Hub
      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./http-server
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-httpd:latest
          # Build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

```
---
# TP3

connect ssh :
- chmod 400 id_rsa
- ssh -i id_rsa admin@emre.elma.takima.cloud

add /etc/ansible/hosts  -> add emre.elma.takima.cloud

HTML affiche http://emre.elma.takima.cloud/


ansible all -i ansible/inventories/setup.yml -m ping
ansible-playbook -i inventories/setup.yml ansible/playbook.yml
ansible-playbook -i inventories/setup.yml ansible/playbook.yml--syntax-check

connect ssh and test docker: systemctl status docker

add roles :docker:
```yml
- hosts: all
  gather_facts: true
  become: true

  roles:
    - docker

  tasks:
    # Install prerequisites for Docker
    - name: Install required packages
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
          - python3-venv
        state: latest
        update_cache: yes

    # Add Docker’s official GPG key
    - name: Add Docker GPG key
      apt_key:
        url: https://download.docker.com/linux/debian/gpg
        state: present

    # Set up the Docker stable repository
    - name: Add Docker APT repository
      apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/debian {{ ansible_facts['distribution_release'] }} stable"
        state: present
        update_cache: yes

    # Install Docker
    - name: Install Docker
      apt:
        name: docker-ce
        state: present

    # Install Python3 and pip3
    - name: Install Python3 and pip3
      apt:
        name:
          - python3
          - python3-pip
        state: present

    # Create a virtual environment for Python packages
    - name: Create a virtual environment for Docker SDK
      command: python3 -m venv /opt/docker_venv
      args:
        creates: /opt/docker_venv  # Only runs if this directory doesn’t exist

    # Install Docker SDK for Python in the virtual environment
    - name: Install Docker SDK for Python in virtual environment
      command: /opt/docker_venv/bin/pip install docker

    # Ensure Docker is running
    - name: Make sure Docker is running
      service:
        name: docker
        state: started
      tags: docker
```
ansible-galaxy init ansible/roles/copy_env_file
ansible-galaxy init ansible/roles/install_docker
ansible-galaxy init ansible/roles/create_network
ansible-galaxy init ansible/roles/launch_database
ansible-galaxy init ansible/roles/launch_app
ansible-galaxy init ansible/roles/launch_proxy

ne garder que tasks et handlers, chaque rôle va avoir une tâche

Le front sera aussi accessible sur:
http://emre.elma.takima.cloud/

ansible-playbook -i ansible/inventories/setup.yml ansible/playbook.yml




