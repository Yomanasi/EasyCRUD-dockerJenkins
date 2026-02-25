# 🚀 EasyCRUD Dockerized Project with CI/CD (Custom Jenkins Pipeline)

This project demonstrates a complete CI/CD pipeline using:

* ✅ Dockerized Frontend & Backend
* ✅ MariaDB Database (RDS)
* ✅ Jenkins Pipeline (Custom)
* ✅ Docker Hub Integration
* ✅ Deployment on AWS EC2 (Ubuntu)
---

# 🧱 Architecture Overview

* **Frontend** → Docker → Port 80
* **Backend (Spring Boot)** → Docker → Port 8080
* **Database** → MariaDB → Port 3306
* **CI/CD Tool** → Jenkins (Running on Port 8081)
* **Images** → Pushed to Docker Hub
* **Server** → AWS EC2 (Ubuntu)

---

# ✅ Prerequisites

* AWS EC2 instance (Ubuntu)
* Docker Hub account
* Security Group open ports:

  * 80 (Frontend)
  * 8080 (Backend)
  * 8081 (Jenkins)
  * 3306 (MariaDB)

---

# ⚙️ Server Setup on EC2

---

## 1️⃣ Install Java 17 (Required for Jenkins)

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
java -version
```

---

## 2️⃣ Install Jenkins

```bash

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins
```

---

## 3️⃣ 🔥 Change Jenkins Default Port to 8081

By default, Jenkins runs on port 8080.
In this project, Jenkins was configured to run on **port 8081**.

Edit Jenkins service file:

```bash
sudo nano /lib/systemd/system/jenkins.service
```

Locate the following line:

```
Environment="JENKINS_PORT=8080"
```

Change it to:

```
Environment="JENKINS_PORT=8081"
```

Then reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

Access Jenkins:

```
http://<EC2-PUBLIC-IP>:8081
```

Get initial admin password:

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

## 4️⃣ Install Docker

```bash
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

---

## 5️⃣ Grant Jenkins Sudo Privileges (Optional but Used Here)

```bash
sudo visudo
jenkins ALL=(ALL) NOPASSWD: ALL
```

---

## 6️⃣ Install MariaDB

```bash
sudo apt update
sudo apt install mysql-client -y
mysql -h <RDS-ENDPOINT> -u <USERNAME> -p
```

---

# 🗄️ Database Configuration

Inside MySQL:
```
CREATE DATABASE student_db;
GRANT ALL PRIVILEGES ON springbackend.* TO 'username'@'localhost' IDENTIFIED BY 'your_password';
```
```
USE student_db;

CREATE TABLE `students` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `email` varchar(255) DEFAULT NULL,
  `course` varchar(255) DEFAULT NULL,
  `student_class` varchar(255) DEFAULT NULL,
  `percentage` double DEFAULT NULL,
  `branch` varchar(255) DEFAULT NULL,
  `mobile_number` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=80 DEFAULT CHARSET=latin1 COLLATE=latin1_swedish_ci;

EXIT;
```

Ensure port 3306 is open in the Security Group.

---

# 🧾 Backend Configuration

Update `application.properties`:

```
server.port=8081

spring.datasource.url=jdbc:mysql://<EC2-IP>:3306/student_db
spring.datasource.username=admin
spring.datasource.password=admin321

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MariaDBDialect
```

---

# 🌐 Frontend Configuration

Update `.env` file:

```
VITE_API_URL="http://<EC2-IP>:8081/api"
```

---

# 🐳 Docker Cleanup Command (Used in Project)

```bash
docker kill $(docker ps -q) && \
docker rm -v $(docker ps -a -q) && \
docker rmi $(docker images -q)
```

---

# 🔐 Store DockerHub Credentials in Jenkins

Go to:

```
Jenkins Dashboard → Manage Jenkins → Manage Credentials
```

Add Credentials:

* Type: Username & Password
* Username: `<dockerhub-username>`
* Password: `<dockerhub-password>`
* ID: `dockerhub-cred`

---

# 🔁 Custom Jenkins Pipeline (Your Implementation)

This is the custom pipeline created for this project.

```groovy
pipeline {
    agent any

    options {
        skipStagesAfterUnstable()
    }

    stages {

        stage('Clone Repo') {
            steps {
                git branch: 'main',
                url: 'https://github.com/Yomanasi/EasyCRUD-Updated.git'
            }
        }
        
        stage('Build Backend Image') {
            steps {
                sh '''
                cd backend
                docker rmi -f $(docker images -q) || true
                docker build -t mimanasi/backend:b53 .
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                cd frontend
                docker build -t mimanasi/frontend:b53 .
                '''
            }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([usernamePassword( credentialsId: 'dockerhub-cred', passwordVariable: 'PASSWORDID', usernameVariable: 'USERNAMEID' )]) {
                    sh '''
                    echo $PASSWORDID | docker login -u $USERNAMEID --password-stdin
                    '''
                }
            }
        }

        stage('Push Backend Image') {
            steps {
                sh '''
                cd backend
                docker push mimanasi/backend:b53
                '''
            }
        }

        stage('Push Frontend Image') {
            steps {
                sh '''
                cd frontend
                docker push mimanasi/frontend:b53
                '''
            }
        }

        stage('Run Backend Container') {
            steps {
                sh '''
                cd backend
                docker rm -f easycrud-backend || true
                docker run -d --name easycrud-backend -p 8080:8080 mimanasi/backend:b53
                '''
            }
        }

        stage('Run Frontend Container') {
            steps {
                sh '''
                cd frontend
                docker rm -f easycrud-frontend || true
                docker run -d --name easycrud-frontend -p 80:80 mimanasi/frontend:b53
                '''
            }
        }
    }
}
```

---

# 🌍 Application Access

Frontend:

```
http://<EC2-IP>
```

Backend:

```
http://<EC2-IP>:8080
```

Jenkins:

```
http://<EC2-IP>:8081
```

⚠️ Important: Since Jenkins and backend both use port 8081, ensure they are not running simultaneously on the same port OR use different ports in production.

---

# 📌 What This CI/CD Pipeline Does

1. Clones repository from GitHub
2. Builds backend Docker image
3. Builds frontend Docker image
4. Logs in to Docker Hub securely
5. Pushes images to Docker Hub
6. Stops existing containers
7. Deploys updated containers

# Fully automated deployment pipeline 🚀

