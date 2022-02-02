# Devops
# TP01 - Docker


# Database

 

On crée une image a partir de postgres pour utiliser une base de donnée. 
On crée un fichier Dockerfile :

    FROM postgres:11.6-alpine
    ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd
**docker build -t tbruendetcpe/mypostgrestp .**
On run aussi un docker adminer pour gerer la BDD 

Pour connecter les deux dockers, on crée un network:
**docker network create myapp**
On run ensuite les dockers:
**docker run -d -p 8080:8080 --network=myapp adminer**
**docker run -p 5432:5432 -e POSTGRES_PASSWORD=pwd --network=host  --name mypostgres tbruendetcpe/mypostgrestp**

le -e permet d'éviter d'écrire dans le Dockerfile le mot de passe en dur et permet de modifier des paramètre simplement

## Init Database
Pour mettre des données dans notre BDD, on doit copier les scripts SQL dans le dossier entrypoint du docker qui est créer automatiquement lors de la création du container

    CREATE TABLE public.departments
    (
    id SERIAL PRIMARY KEY,
    name VARCHAR(20) NOT NULL
    );
    CREATE TABLE public.students
    (
    id SERIAL PRIMARY KEY,
    department_id INT NOT NULL REFERENCES departments (id),
    first_name VARCHAR(20) NOT NULL,
    last_name VARCHAR(20) NOT NULL
    );

On modifie donc le Dockerfile puis on rebuild l'image du docker:

    FROM postgres:11.6-alpine
    
    ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd
    COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
    COPY 02-InsertData.sql /docker-entrypoint-initdb.d

On peut voir dans adminer que les tables sont crées au lancement du docker.

Pour se connecter sur notre bdd, il faut récupérer l'ip du network que l'on a crée:
**docker network inspect myapp**
## Persist Database
Pour garder l'état de la BDD même quand le docker est détruit, il faut créer un volume 
**docker volume create volpostgres** 
On doit ensuite mapper le volume crée au docker quand on le run
**docker run -p 5432:5432 -v volpostgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=pwd --network=myapp  --name mypostgres tbruendetcpe/mypostgrestp**

On peut voir que si on crée une table et redémarre le docker, la table est encore présente.
# Backend API

On crée un fichier Main.java que l'on compile.
On crée une image pour lancer ce java:
**dockerfile**:

    FROM openjdk:11
    RUN mkdir /usr/src/myapp
    COPY Main.class /usr/src/myapp
    WORKDIR /usr/src/myapp
    CMD ["java", "Main"]

**docker build -t  mybackendapi .**
**docker run mybackendapi**

Hello world s'affiche.

## Multiusage build
Pour pouvoir build et run le code java sur notre doker on crée un dockerfile:
Le premier FROM avec Maven permet de build et le deuxième permet de run le code avec une image plus légère. 

    # Build
    FROM maven:3.6.3-jdk-11 AS myapp-build
    ENV MYAPP_HOME /opt/myapp
    WORKDIR $MYAPP_HOME
    COPY pom.xml .
    COPY src ./src
    RUN mvn package -DskipTests
    # Run
    FROM openjdk:11-jre
    ENV MYAPP_HOME /opt/myapp
    WORKDIR $MYAPP_HOME
    COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
    ENTRYPOINT java -jar myapp.jar
La première partie permet de build le java, on part de l'image de maven.
ENV prépare une variable d’environnement 
WORKDIR définit le répertoire de travail
On copie pom.xml et le dossier src dans le docker
RUN nvm package compile le java dans un jar.

La parie Run s'occupe de lancer le jar.

## Backend API
On garde le même dockerfile et on modifie application.yml:

 
  

    jpa:
        properties:
          hibernate:
            jdbc:
              lob:
                non_contextual_creation: true
        generate-ddl: false
        open-in-view: true
      datasource:
        url: jdbc:postgresql://mypostgres:5432/db
        username: usr
        password: pwd
        driver-class-name: org.postgresql.Driver
    management:
     server:
       add-application-context-header: false
     endpoints:
       web:
         exposure:
           include: health,info,env,metrics,beans,configprops

On lance les deux docker sur le même network:
**docker run -p 5432:5432 -e POSTGRES_PASSWORD=pwd --network=myapp  --name mypostgres mypostgres**

**docker run -p 8080:8080 --network=myapp mybackend**

# HTTP
On build une image d'apache :

    FROM httpd:2.4.52-alpine
    COPY ./public-html/ /usr/local/apache2/htdocs/
**docker run -d -p 80:80 http**
On peut voir la page html sur localhost:80

## Configuration
On récupère le fichier configuration
**docker cp 94271959ba18fd0799c0830c91052e4454b0aa7663a602bf92233e0f444f1cdb:/usr/local/apache2/conf/httpd.conf .**
## Reverse proxy
On modifie le fichier de configuration pour réaliser un proxy de apache vers notre api. on redirige le flux du port 80 vers le port 8080.
On ajoute au .conf :

    <VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://mybackend:8080/
    ProxyPassReverse / http://mybackend:8080/
    </VirtualHost>

On copie le .conf dans l'image apache:

    FROM httpd:2.4.52-alpine
    COPY ./public-html/ /usr/local/apache2/htdocs/
    COPY httpd.conf /usr/local/apache2/conf/httpd.conf

On lance tous les docker sur le même network:

**docker run -p 8080:8080 --name mybackend --network=myapp mybackend**

**docker run --network=myapp -p 80:80 http**

**docker run -p 5432:5432 -e POSTGRES_PASSWORD=pwd --network=myapp  --name mypostgres mypostgres**
## Compose 
Pour lancer les 3 docker en une commande on utilise docker compose.
On crée un fichier docker_compose.yml :

    version: '2.0'
    services:
      
      mybackend: 
        build:
          ./backendapi/simple-api
        networks:
          - myapp
        depends_on:
          - mypostgres
    
      mypostgres:
        build:
          ./postgres
        networks:
          - myapp
        volumes:
          - volpostgres:/var/lib/postgresql/data
        
      httpd:
        build:
          ./http
        ports:
          - "80:80"
        networks:
          - myapp
        depends_on:
          - mypostgres
          - mybackend
    
    networks:
      myapp:
    
    volumes:
      volpostgres: {}

Puis on lance la commande:
**docker-compose up -d**
Pour le fermer:
**docker-compose rm**

On peut fermer un seul des containers au besoin avec
**docker-compose rm NomContainer**

## Publish

On peut déposer nos images sur le dockerhub
**docker tag my-database USERNAME/mypostgres:1.0**
Docker tag permet de donner une version à notre image.
Pour la publier, on utilise
**docker push USERNAME/mypostgres**

# TP-02 CI/CD

Pour tester le code, on utilise la commande:
**mvn clean verify**
Elle utilise les testcontainers présent dans le pom.xml. Ce sont des librairies java permettant de lancer des dockers pour tester notre code

On souhaite lancer le test a chaque pull.
On crée un dossier .github/workflow dans le repo git puis on crée le fichier .main.yml:

    name: CI devops 2022 CPE
    
    on:
    
    #to begin you want to launch this job in main and develop
    
    push:
    
    branches:
    
    - main
    
    - develop
    
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
    
    java-version: '11'
    
    distribution: 'adopt'
    
    #finally build your app with the latest command
    
    - name: Build and test with Maven
    
    run: mvn clean verify --file ./backendapi/simple-api
On peut ensuite voir sur github dans l'onglet action les workflows, la validation est alors validé ou non.

On modifie le .main pour pouvoir push nos images docker sur notre compte dockerhub:

 < # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest
    
    # steps to perform in jobe
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Build image and push backend
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./backendapi/simple-api
          # Note: tags has to be all lower-case
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-cpe:mybackend
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./postgres
          # Note: tags has to be all lower-case
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-cpe:mypostgres
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}
        # DO the same for database
      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./http
          # Note: tags has to be all lower-case
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-cpe:http
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }} >

Pour des raisons de sécurité, on déclare des variables d'actions secrètes sur github (DOCKERHUB_TOKEN et DOCKERHUB_USERNAME) que l'on utilise avec  **${{ secrets.DOCKERHUB_USERNAME }}**

A chaque push on peut vérifier que l'image docker a bien été push sur le dockerhub dans l'onglet repositories.

## Sonarcloud

Pour s'assurer de la qualité du code et de sa sécurité, on utilise un sonar qui va analyser le code à chaque push.
On crée d'abord un compte sonarcloud que l'on lie à Github. Puis on suis les indication de sonarcloud pour configurer cette vérification. 
On rajoute dans le pom.yml les properties pour sonarcloud:

 	<properties>
		<java.version>11</java.version>
		<jacoco-maven-plugin.version>0.8.6</jacoco-maven-plugin.version>
		<sonar.organization>theo-bruendet</sonar.organization>
  		<sonar.host.url>https://sonarcloud.io</sonar.host.url>
	</properties>

On modifie aussi le .main:


    on:
      #to begin you want to launch this job in main and develop
      push:
        branches: 
          - main
          - develop
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
          java-version: '11'
          distribution: 'adopt'
      #finally build your app with the latest command
      - name: Build and test with Maven
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Needed to get PR information, if any
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=Theo-Bruendet_Devops --file ./backendapi/simple-api/pom.xml

Maintenant à chaque push on peut voir sur le site sonarcloud le résultat de chaque test et les potentiel chose à modifier pour améliorer le code.

# TP-03 Ansible
## Inventory

On crée un fichier .yml pour rentrer les données du serveur et lancer ansible avec ces cofiguration:

    all:
     vars:
      ansible_user: centos
      ansible_ssh_private_key_file: /fs03/share/users/theo.bruendet/home/Documents/id_rsa
     children:
      prod:
       hosts: theo.bruendet.takima.cloud

On lance ensuite la commande :**$ ansible all -i inventories/setup.yml -m ping**

    theo.bruendet@tpc28:~/Documents/Devopsgit/Devops$ ansible all -i inventories/setup.yml -m ping
    theo.bruendet.takima.cloud | SUCCESS => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/libexec/platform-python"
        }, 
        "changed": false, 
        "ping": "pong"
    }

Le serveur répond au ping.

## Facts

On récupère les variable d'information de l’hôte avec : **ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution"**

    theo.bruendet.takima.cloud | SUCCESS => {
        "ansible_facts": {
            "ansible_distribution": "CentOS", 
            "ansible_distribution_file_parsed": true, 
            "ansible_distribution_file_path": "/etc/redhat-release", 
            "ansible_distribution_file_variety": "RedHat", 
            "ansible_distribution_major_version": "8", 
            "ansible_distribution_release": "NA", 
            "ansible_distribution_version": "8", 
            "discovered_interpreter_python": "/usr/libexec/platform-python"
        }, 
        "changed": false
    }
On remove le serveur apache installer lors du td :
**$ ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become**

On crée notre playbook:

    - hosts: all
      gather_facts: false
      become: yes
    
      tasks:
      - name: Test connection
        ping:
    
   On le lance avec: **ansible-playbook -i inventories/setup.yml playbook.yml**
    

    PLAY [all] *********************************************************************
    
    TASK [Test connection] *********************************************************
    ok: [theo.bruendet.takima.cloud]
    
    PLAY RECAP *********************************************************************
    theo.bruendet.takima.cloud : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  

## Advance playbook

On crée un nouveau playbook pour installer docker:

    - hosts: all
      gather_facts: false
      become: yes
    # Install Docker
      tasks:
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
   
   On le lance avec: **ansible-playbook -i inventories/setup.yml playbook.yml**

       PLAY [all] **********************************************************************************************************************************
    
    TASK [Clean packages] ***********************************************************************************************************************
    [WARNING]: Consider using the dnf module rather than running 'dnf'.  If you need to use command because dnf is insufficient you can add
    'warn: false' to this command task or set 'command_warnings=False' in ansible.cfg to get rid of this message.
    changed: [theo.bruendet.takima.cloud]
    
    TASK [Install device-mapper-persistent-data] ************************************************************************************************
    changed: [theo.bruendet.takima.cloud]
    
    TASK [Install lvm2] *************************************************************************************************************************
    changed: [theo.bruendet.takima.cloud]
    
    TASK [add repo docker] **********************************************************************************************************************
    [WARNING]: Consider using 'become', 'become_method', and 'become_user' rather than running sudo
    changed: [theo.bruendet.takima.cloud]
    
    TASK [Install Docker] ***********************************************************************************************************************
    changed: [theo.bruendet.takima.cloud]
    
    TASK [install python3] **********************************************************************************************************************
    changed: [theo.bruendet.takima.cloud]
    
    TASK [Pip install] **************************************************************************************************************************
    changed: [theo.bruendet.takima.cloud]
    
    TASK [Make sure Docker is running] **********************************************************************************************************
    changed: [theo.bruendet.takima.cloud]
    
    PLAY RECAP **********************************************************************************************************************************
    theo.bruendet.takima.cloud : ok=8    changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 

On peut voir les différentes taches qui s’exécute et que les 8 sont terminées.

## Using role

On initialise le dossier qui va contenir notre role docker avec la commande:
**$ ansible-galaxy init roles/docker**
On déplace la task du playbook vers le main.yml du rôle docker:
**main:**

     main:
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

**playbook:**

    - hosts: all
      gather_facts: false
      become: yes
    # Install Docker
      roles:
        - docker
## Deploy your app

Pour déployer notre api sur le serveur on crée un rôle pour chaque docker
**Proxy**

    - name: Run proxy
      docker_container:
        name: http
        image: tbruendetcpe/tp-devops-cpe:http
        ports:
          - "80:80"
        networks:
          - name: network_one

**backend**

    - name: Run app
      docker_container:
        name: mybackend
        image: tbruendetcpe/tp-devops-cpe:mybackend
        networks:
          - name: network_one
**database**

    - name: Run database
      docker_container:
        name: mypostgres
        image: tbruendetcpe/tp-devops-cpe:mypostgres
        networks:
          - name: network_one
On crée un rôle pour la création du network

    - name: Create network
      docker_network:
        name: network_one
On lance ensuite tous ces fichiers à partir du playbook:

    - hosts: all
      gather_facts: false
      become: yes
    # Install Docker
      roles:
        - docker
        - network
        - app
        - database
        - proxy



