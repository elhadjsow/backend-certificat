pipeline {
    agent any

    environment {
        IMAGE_NAME = 'elhadjsow/backend-certificat'
        IMAGE_TAG = 'latest'
        POSTGRES_CONTAINER = 'postgres_test'
        POSTGRES_PORT = '5433'
        POSTGRES_DB = 'certificatdb'
        POSTGRES_USER = 'postgres'
        POSTGRES_PASSWORD = '1234'
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
                powershell """
                docker run -d --name ${POSTGRES_CONTAINER} `
                    -e POSTGRES_DB=${POSTGRES_DB} `
                    -e POSTGRES_USER=${POSTGRES_USER} `
                    -e POSTGRES_PASSWORD=${POSTGRES_PASSWORD} `
                    -p ${POSTGRES_PORT}:5432 postgres:17
                Start-Sleep -Seconds 5
                """
            }
        }

        stage('Run Tests') {
            steps {
                echo '🧪 Lancement des tests unitaires...'
                powershell """
                \$env:POSTGRES_HOST = "localhost"
                \$env:POSTGRES_PORT = "${POSTGRES_PORT}"
                \$env:POSTGRES_DB = "${POSTGRES_DB}"
                \$env:POSTGRES_USER = "${POSTGRES_USER}"
                \$env:POSTGRES_PASSWORD = "${POSTGRES_PASSWORD}"
                python manage.py test
                """
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
                withCredentials([usernamePassword(credentialsId: 'docker-hub-pat', usernameVariable: 'DOCKER_HUB_USR', passwordVariable: 'DOCKER_HUB_PSW')]) {
                    powershell '''
$dockerDir = "$env:USERPROFILE\\.docker"
if (!(Test-Path $dockerDir)) {
    New-Item -ItemType Directory -Path $dockerDir -Force | Out-Null
}

$authString = "$env:DOCKER_HUB_USR" + ":" + "$env:DOCKER_HUB_PSW"
$authBase64 = [Convert]::ToBase64String([System.Text.Encoding]::ASCII.GetBytes($authString))

$configContent = @"
{
    "auths": {
        "https://index.docker.io/v1/": {
            "auth": "$authBase64"
        }
    }
}
"@

$configContent | Out-File -FilePath "$dockerDir\config.json" -Encoding UTF8 -Force
Write-Host "Push de l'image..."
docker push "$env:IMAGE_NAME`:$env:IMAGE_TAG"
if ($LASTEXITCODE -eq 0) {
    Write-Host "Image poussee!"
    Remove-Item "$dockerDir\config.json" -Force -ErrorAction SilentlyContinue
} else {
    Write-Host "Erreur push"
    exit 1
}
'''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo '🚀 Déploiement avec Docker Compose...'
                powershell """
                docker-compose down
                docker-compose up -d
                """
            }
        }
    }

    post {
        always {
            echo 'Arrêt et nettoyage des conteneurs de test...'
            powershell """
            docker stop ${POSTGRES_CONTAINER} 2>$null
            docker rm ${POSTGRES_CONTAINER} 2>$null
            """
            echo 'Pipeline terminé ✅'
        }
        failure {
            echo '❌ Le pipeline a échoué'
        }
    }
}
