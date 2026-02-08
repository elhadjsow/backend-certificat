pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub-credentials')
        IMAGE_NAME = 'elhadjsow/backend-certificat'
        IMAGE_TAG = 'latest'
    }

    stages {
        stage('Checkout') {
            steps {
                echo '📥 Clonage du code source...'
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                echo '📦 Installation des dépendances...'
                bat 'pip install -r requirements.txt'
            }
        }

        stage('Start PostgreSQL') {
            steps {
                echo '🗄️ Démarrage de PostgreSQL...'
                powershell '''
                docker run -d --name postgres_test -e POSTGRES_DB=certificatdb -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=1234 -p 5433:5432 postgres:17
                Start-Sleep -Seconds 5
                '''
            }
        }

        stage('Run Tests') {
            steps {
                echo '🧪 Lancement des tests unitaires...'
                powershell '''
                $env:POSTGRES_HOST = "localhost"
                $env:POSTGRES_PORT = "5433"
                $env:POSTGRES_DB = "certificatdb"
                $env:POSTGRES_USER = "postgres"
                $env:POSTGRES_PASSWORD = "1234"
                python manage.py test
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '🐳 Construction de l’image Docker...'
                bat 'docker build -t %IMAGE_NAME%:%IMAGE_TAG% .'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo '📤 Envoi de l’image vers Docker Hub...'
                powershell '''
                $password = "$env:DOCKER_HUB_CREDENTIALS_PSW"
                $username = "$env:DOCKER_HUB_CREDENTIALS_USR"
                $imageName = "$env:IMAGE_NAME"
                $imageTag = "$env:IMAGE_TAG"
                
                # Login to Docker Hub
                echo $password | docker login -u $username --password-stdin
                
                # Push image
                docker push "$imageName:$imageTag"
                docker logout
                '''
            }
        }
        stage('Deploy') {
            steps {
                echo '🚀 Déploiement avec Docker Compose...'
                powershell '''
                docker pull $env:IMAGE_NAME:$env:IMAGE_TAG
                docker-compose down
                docker-compose up -d
                '''
            }
        }
    }

    post {
        always {
            echo 'Arrêt et nettoyage des conteneurs de test...'
            powershell '''
            docker stop postgres_test 2>$null
            docker rm postgres_test 2>$null
            '''
            echo 'Pipeline terminé ✅'
        }
        failure {
            echo '❌ Le pipeline a échoué'
        }
    }
}
