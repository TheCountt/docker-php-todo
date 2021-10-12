# PHP-TODO Application Containerization using Docker

1. Download or clone php-todo repository using `wget` or `git clone`
2. Write a dockerfile for php-todo app and save it in the php-todo directory
```
FROM php:7.4.24-apache
LABEL Dare=dare@zooto.io

#install zip, unzip extension, git, mysql-client
RUN apt-get update --fix-missing && apt-get install -y \
  default-mysql-client \
  git \
  unzip \
  zip \
  curl \
  wget
  
# Install docker php dependencies
RUN docker-php-ext-install pdo_mysql mysqli

# Add config files and binary file and enable webserver
COPY apache-config.conf /etc/apache2/sites-available/000-default.conf
COPY start-apache /usr/local/bin
RUN a2enmod rewrite

RUN curl -sS https://getcomposer.org/installer |php && mv composer.phar /usr/local/bin/composer

# Copy application source
COPY . /var/www
RUN chown -R www-data:www-data /var/www

EXPOSE 80

CMD ["start-apache"]
```

4. Create a docker-compose.yml in the php-todo directory and paste the code below:

```
version: "3.9"
services:
 app:
    build:
      context: .
    container_name: php-website
    network_mode: tooling_app_network
    restart: unless-stopped
    volumes:
      - app:/php-todo
    ports:
      - "${APP_PORT}:80"
    depends_on:
      - db

 db:
    image: mysql/mysql-server:latest
    container_name: php-db-server
    network_mode: tooling_app_network
    hostname: "${DB_HOSTNAME}"
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: "${DB_DATABASE}"
      MYSQL_USER: "${DB_USERNAME}"
      MYSQL_PASSWORD: "${DB_PASSWORD}"
      MYSQL_ROOT_PASSWORD: "${DB_ROOT_PASSWORD}"
    
    ports:
      - "${DB_PORT}:3306"

    volumes:
      - db:/var/lib/mysql

volumes:
  app:
  db:
  ```

4. Update the `.env` file
```
APP_PORT=8000
APP_ENV=local
APP_DEBUG=true
APP_KEY=SomeRandomString
APP_URL=http://localhost

DB_HOSTNAME=mysqlserverhost
DB_DATABASE=homestead
DB_USERNAME=homestead
DB_PASSWORD=sePret^1
DB_ROOT_PASSWORD=Qwerty123@
DB_PORT=3306

CACHE_DRIVER=file
SESSION_DRIVER=file
QUEUE_DRIVER=sync

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_DRIVER=smtp
MAIL_HOST=mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
```

- Make sure you change directory to php-todo directory. Build image using this command:
```
docker build -t php-todo:latest .
```

- Make sure you change directory to php-todo directory. Deploy the containers:
```
docker-compose up -d
```
- We are going to open a docker hub account if we do not have already. Go to  your bbroswer and open a dockerhub account
- On your terminal/editor, create a new tag for the image you want to push using the
proper syntax.

```
docker tag php-todo:latest thecountt/php-todo:1.0.0
```

- Run this command to see the image with the newly created tag

```
docker ps -a
```

Login to your dockerhub account and type in your credentials

```
docker login
```

- Push the docker image from the local machine to the dockerhub repository
```
docker push thecountt/php-todo:1.0.0
```

## CI/CD with Jenkins (Container or Machine)

### 1. Using Local Machine

- Run the following command in your home directory to install java runtime:
```
sudo apt update -y
sudo apt install openjdk-11-jdk
```
- Run the following commands to install jenkins:
```
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
```

#### Unlocking Jenkins
- When you first access a new Jenkins instance, you are asked to unlock it using an automatically-generated password.

- Browse to http://localhost:8080 (or whichever port you configured for Jenkins when installing it) and wait until the Unlock Jenkins page appears and you can use

`sudo cat /var/lib/jenkins/secrets/initialAdminPassword` to print the password on the terminal.


#### Jenkins Pipeline
- First we will install the plugins needed
  - On the Jenkins Dashboard, click on `Manage Jenkins` and go to `Manage Plugins`.
  - Search and install the following plugins:
    - Blue Ocean
    - Docker
    - Docker Compose Build Steps
    - HttpRequest
    

- We need to create credentials that we will reference so as to be able to push our image to the docker hub repository

  - Click on  `Manage Jenkins` and go to `Manage Credentials`. Click on `global` and create a

- Create a new repository in your dockerhub account to push image into

- Create a Jenkinsfile in the php-todo directory. 

### 2. Using Docker Container
- Create a directory and name it `jenkins` and change into the directory.
- Create a bridge network in Docker using the following docker network create command. We will use the network we have created earlier(tooling_app_network)

- In order to execute Docker commands inside Jenkins nodes, download  the `docker:dind` Docker image so we can use it in our docker-compose file we will create:

```
docker pull docker:dind
```

- Customise official Jenkins Docker image, by executing the two steps below:

    a. Create Dockerfile with the following content:

```
FROM jenkins/jenkins:2.303.1-jdk11
USER root
RUN apt-get update && apt-get install -y apt-transport-https \
       ca-certificates curl gnupg2 \
       software-properties-common
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
RUN apt-key fingerprint 0EBFCD88
RUN add-apt-repository \
       "deb [arch=amd64] https://download.docker.com/linux/debian \
       $(lsb_release -cs) stable"
RUN curl -L \  
  "https://github.com/docker/compose/releases/download/v2.0.0-beta.6/docker-compose-$(uname -s)-$(uname -m)" \  
  -o /usr/local/bin/docker-compose \  
  && chmod +x /usr/local/bin/docker-compose
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.25.0 docker-workflow:1.26"

```

   b. Build a new docker image from this Dockerfile and assign the image a meaningful name, e.g. "myjenkins-blueocean:1.1":
```
docker build -t myjenkins-blueocean:1.1 . 
```
- Create a file and name it jenkins.yml. Paste the code below:

```
version: "3.9"
services:
    docker:
        image: "docker:dind"
        container_name: jenkins-docker
        privileged: true
        network_mode: tooling_app_network
        
        environment:
            - DOCKER_TLS_CERTDIR=/certs
            - DOCKER_DRIVER=overlay2
        volumes:
            - 'jenkins-docker-certs:/certs/client'
            - 'jenkins-data:/var/jenkins_home'
        ports:
            - "2376:2376"

    myjenkins-blueocean:
        image: "myjenkins-blueocean:1.1"
        container_name: jenkins-blueocean
        network_mode: tooling_app_network
        environment:
            - DOCKER_HOST=tcp://docker:2376
            - DOCKER_CERT_PATH=/certs/client
            - DOCKER_TLS_VERIFY=1
        ports:
            - "8080:8080"
            - "50000:50000"
        
        volumes:
            - "jenkins-data:/var/jenkins_home"
            - "jenkins-docker-certs:/certs/client:ro"
        
volumes:
   docker:
   myjenkins-blueocean:
   jenkins-docker-certs:
   jenkins-data:
```
- Spin up jenkins containers:
```
docker-compose -f jenkins.yml up -d
```

### Unlocking Jenkins
- When you first access a new Jenkins instance, you are asked to unlock it using an automatically-generated password.

- Browse to http://localhost:8080 (or whichever port you configured for Jenkins when installing it) and wait until the Unlock Jenkins page appears

- If you are running Jenkins in Docker using the official jenkins/jenkins image you can use:

`sudo docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword` to print the password in the console without having to exec into the container.


### Jenkins Pipeline

- Create a new branch from our main branch in your github repo and name it "feature". So we have two github branches: main and feature

- Create a new repository in your dockerhub account to push image into

- Create a Jenkinsfile in the php-todo directory. Write a Jenkinsfile that will pick up the build context from the referenced github repo branch, simulate a Docker Build and a
Docker Push to the dockerhub registry.


```
pipeline {
    environment {
        registry = "thecountt/docker-php-todo"
        registryCredential = 'docker-hub-cred'
    }
    agent any
    stages {
        
        stage('Cloning Git repository') {
          steps {
                git branch : 'main', url: 'https://github.com/TheCountt/docker-php-todo.git'
            }
        }

        stage('Build Image') {
            steps{
                script {
                    dockerImage = docker.build registry + ":$BUILD_NUMBER"
                }
            }
        }

        stage('Push Image') {
            steps{
                script {
                    docker.withRegistry( '', registryCredential ) {
                        dockerImage.push()
                    }
                }
            }
        }

        stage('Remove Unused docker image') {
            steps{
                sh "docker rmi $registry:$BUILD_NUMBER"
            }
        }

    }
}
```


- Create a Multibranch Pipeline using Blue Ocean.

- If you run a build, the pipeline will fail because we have not configured our dockerhub
(registryCredential) in jenkins.
  a. Go to jenkins home, click on your pipeline. On the left-hand side, click on "configure"
  b. Click on "Properties".
  c. Fill the "Docker registry URL"
  c. In the "Registry credentials", click on "Add". Put your dockerhub account credentials
     (Username and Password) and save it.

- Click on "Scan Repository"

- Run a build now on each pipeline now. The two pipelines should be successful

- Create a github webhook so jenkins can automatically pickup changes and run a build.

