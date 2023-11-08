# DevOps - Rendu TP


## TP part 01- Docker
### 1-1 Dabatase
```DockerFile
FROM postgres:16-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=user \
    POSTGRES_PASSWORD=pwd

COPY *.sql /docker-entrypoint-initdb.d
```

Dans cette section, nous mettons en place la configuration initiale de notre conteneur de base de données. 

La première commande nous permet de spécifier à Docker l'image requise pour notre conteneur. 
Les lignes liées à *ENV* servent à définir les paramètres de notre base de données, notamment son nom, le nom d'utilisateur et le mot de passe associé. Enfin, *COPY* nous permet d'intégrer nos fichiers SQL contenus dans le dossier .sql dans le conteneur.

### Commandes
Voici les commandes saisies pour lancer nos containers:
```Command
docker network create app-network

#Crée un réseau Docker nommé "app-network".

docker build -t guibdrd/databasecontainer .

#Construit une image Docker nommée "guibdrd/databasecontainer" à partir du Dockerfile

docker run -p "5432:5432" --net=app-network --name=database -d guibdrd/databasecontainer

#Lance un conteneur de base de données en mapant le port 5432, connecté au réseau "app-network".

docker run -p "8090:8080" --net=app-network --name=adminer -d adminer

#Lance un conteneur "adminer" en mapant le port 8080, connecté au même réseau "app-network".

```

## Backend API
### 1-2 Multistage build
Un multistage build permet d'optimiser au maximum l'image Docker, en réduisant alors sa taille en y retirant les dépendances et informations inutiles.

#### Dockerfile:
```dockerfile
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```
##### Build
FROM : construction d'une image Maven que l'on nomme "myapp-build"
ENV : définition de la variable d'environnement
WORKDIR: répertoir de travail
COPY: Copie du pom.xml et du répertoire src vers le conteneur
RUN; Éxécution de la commande "mvn package -DskipTests" pour construire l'application en ignorant les tests unitaires.

##### Run
FROM : utilisation de l'image Amazon Corretto 17 comme base pour exécuter l'application
ENV : définition de la variable d'environnement
WORKDIR: répertoire de travail
COPY : Copie du fichier JAR généré lors de l'étape de construction (myapp.jar) dans le répertoire de travail de cette étape
ENTRYPOINT : Lancement de l'application en exécutant la commande "java -jar myapp.jar" lors du démarrage du conteneur.


### Link Application
#### 1.3 - Commandes Docker-compose
- docker-compose up: Construit les images et les lances en tant que container.

- docker-compose down: Arrête et supprime les containers ainsi que les volumes et réseaux associés.

- docker-compose ps: Affiche les containers, leur état, leur noms, les ports associés ...

- docker-compose logs: Affiche les logs du container spécifié, pour analyser les erreurs par exemple.

- docker-compose build: Reconstruit les images pour y apporter lesmodifications  ajoutés précédemment.

#### 1.4 - Documente le fichier docker-compose

[Cliquer ici pour accéder au fichier documenté](/TP1-2/docker-compose.yaml)

## TP part 02 - Github-Action

### Build and test your Application

Les Testcontainers sont des conteneurs Docker temporaires qui permettent de faciliter les tests d'intégration et/ou unitaire, utilisés principalement dans le développement logicielle nécessitant des dépendances externes.




```yaml
name: CI devops 2023
on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main
      - develop

jobs:
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: 17
          distribution: "adopt"

      - name: Build and test with Maven
        run: |
          mvn -B verify sonar:sonar -Dsonar.projectKey=devopstp2guillaume_devops -Dsonar.organization=devopstp2guillaume -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./simple-api/pom.xml
```

#### Affichage dans SonarCloud

![Affichage dans SonarCloud](/SonarCloud.png)

Nous utilisons des secured variables afin de stocker les données car ce sont des données sensibles (password / Token) que nous souhaitons par partager avec d'autres utilisateurs.

On utilise "needs: test-backend" afin de lancer le job "build-and-push-docker-image" seulement lorsque le job "test-backend- fonctionne correctement (compile + tests").

![CI fonctionnel et push](/Working%20CI.png)

## TP part 03 - Ansible
Notre répertoire "Inventories" contient un unique fichier: setup.yml

### 3-1 setup.yml:
```yaml
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: ../../../id_rsa
  children:
    prod:
      hosts: guillaume.bodard.takima.cloud
```

- "ansible_user" indique à ansible l'utilisateur
- "hosts" indique l'adresse du serveur
- "ansible_ssh_private_key_file" indique le chemin relatif de la clé ssh privée (ici un fichier nommé id_rsa)


Ces données sont alors inscrites dans le fichier **setup.yml**.
Ainsi, il est possible avec une commande de configurer la connexion au serveur en utilisant les données du fichier ci-dessus.

```command
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

Pour qu'Ansible puisse commencer les configurations, nous avons besoin de créer un *playbook* qui gèrera les *rôles*, automatisant la configuration et les services.

###### 3-2 playbook.yml
```yaml
- hosts: all
  gather_facts: false
  become: yes
  roles:
    - docker
    - network
    - database
    - app
    - proxy
```

Ainsi, le playbook nous permet de lancer avec tous les serveurs (*hosts: all*) les rôles spécifiés dans *roles* (docker, network, etc...) avec une unique commande, et donc d'éviter une répétition de plusieurs commandes.

###### 3-3 docker_container tasks

```yaml
- name: Clean packages
  command:
    cmd: yum clean -y packages

- name: Install  
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker

```

Le script ci-dessus correspond à une série de tâche permettant de configurer l'environnement Docker.

- Nettoyer les packages
- Installe device-mapper-persistent-data via yum
- Installe lvm2 via yum
- Ajoute un répertoire docker qui permettra de le télécharger ensuite
- Installe Docker
- S'assure que Docker est bien lancée et fonctionne correctement
