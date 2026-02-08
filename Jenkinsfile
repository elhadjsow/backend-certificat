pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'elhadjsow/backend-certificat'
        DOCKER_TAG = "${BUILD_NUMBER}"
        POSTGRES_CONTAINER = 'postgres_test'
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
                    # Arrêter et supprimer le conteneur s'il existe
                    $ErrorActionPreference = 'SilentlyContinue'
                    docker stop postgres_test
                    docker rm postgres_test
                    $ErrorActionPreference = 'Continue'
                    
                    # Démarrer un nouveau conteneur PostgreSQL
                    docker run -d `
                        --name postgres_test `
                        -e POSTGRES_USER=testuser `
                        -e POSTGRES_PASSWORD=testpass `
                        -e POSTGRES_DB=testdb `
                        -p 5433:5432 `
                        postgres:13
                    
                    # Attendre que PostgreSQL soit prêt
                    Write-Host "Attente du démarrage de PostgreSQL..."
                    Start-Sleep -Seconds 10
                    
                    # Vérifier que le conteneur est bien démarré
                    docker ps | Select-String postgres_test
                '''
            }
        }
        
        stage('Run Tests') {
            steps {
                echo '🧪 Exécution des tests...'
                bat '''
                    set DATABASE_URL=postgresql://testuser:testpass@localhost:5433/testdb
                    python manage.py test
                '''
            }
        }
        
        stage('Code Quality Check') {
            steps {
                echo '🔍 Vérification de la qualité du code...'
                bat 'flake8 . --max-line-length=120 --exclude=migrations,venv,env'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo '🐳 Construction de l\'image Docker...'
                bat """
                    docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                    docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
                """
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                echo '📤 Publication vers Docker Hub...'
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    bat """
                        docker login -u %DOCKER_USERNAME% -p %DOCKER_PASSWORD%
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }
        
        stage('Deploy') {
            steps {
                echo '🚀 Déploiement de l\'application...'
                powershell '''
                    # Arrêter et supprimer l'ancien conteneur de l'application
                    $ErrorActionPreference = 'SilentlyContinue'
                    docker stop backend-certificat-app
                    docker rm backend-certificat-app
                    $ErrorActionPreference = 'Continue'
                    
                    # Démarrer le nouveau conteneur
                    docker run -d `
                        --name backend-certificat-app `
                        -p 8000:8000 `
                        -e DATABASE_URL=postgresql://testuser:testpass@host.docker.internal:5433/testdb `
                        elhadjsow/backend-certificat:latest
                    
                    Write-Host "Application déployée avec succès!"
                '''
            }
        }
    }
    
    post {
        always {
            echo '🧹 Nettoyage des conteneurs de test...'
            powershell '''
                $ErrorActionPreference = 'SilentlyContinue'
                docker stop postgres_test
                docker rm postgres_test
                $ErrorActionPreference = 'Continue'
                
                Write-Host "Nettoyage terminé"
            '''
        }
        success {
            echo '✅ Pipeline terminé avec succès!'
            echo "Image Docker: ${DOCKER_IMAGE}:${DOCKER_TAG}"
        }
        failure {
            echo '❌ Le pipeline a échoué'
            echo 'Vérifiez les logs ci-dessus pour plus de détails'
        }
    }
}