pipeline {
    agent any
    
    environment {
        GITHUB_USERNAME = 'SalifAbdoulSow18'
        DOCKER_HUB_USERNAME = 'sasow'
        DOCKER_HUB_REPO = 'test'
        IMAGE_NAME = "${DOCKER_HUB_USERNAME}/${DOCKER_HUB_REPO}"
        K8S_NAMESPACE = 'test'
        IP = 'demystops.com' // Remplacez par l'IP de votre cluster ou domaine
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "📦 Récupération du code..."
                git branch: 'main', 
                    url: 'https://github.com/SalifAbdoulSow18/test.git',
                    credentialsId: 'github-credentials'
                echo "✅ Code récupéré"
            }
        }

        stage('Read Version') {
            steps {
                script {
                    // Lire la version actuelle
                    def versionFile = readFile('VERSION').trim()
                    env.APP_VERSION = versionFile
                    echo "📌 Version actuelle: ${env.APP_VERSION}"
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo "🐳 Build de l'image Docker..."
                script {
                    sh """
                        docker build -t ${IMAGE_NAME}:${env.APP_VERSION} .
                        docker tag ${IMAGE_NAME}:${env.APP_VERSION} ${IMAGE_NAME}:latest
                    """
                }
                echo "✅ Image buildée"
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                echo "📤 Push vers Docker Hub..."
                withCredentials([string(credentialsId: 'docker-hub-password', variable: 'DOCKER_PASS')]) {
                    sh """
                        echo "${DOCKER_PASS}" | docker login -u ${DOCKER_HUB_USERNAME} --password-stdin
                        docker push ${IMAGE_NAME}:${env.APP_VERSION}
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                    """
                }
                echo "✅ Image poussée sur Docker Hub"
            }
        }
        
        stage('Update Git Manifests') {
            steps {
                echo "📝 Mise à jour du tag dans les manifests..."
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh """
                        # Mettre à jour l'image dans deployment.yaml avec avec la nouvelle version
                        sed -i "s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${env.APP_VERSION}|" k8s/deployment.yaml
                        
                        git config user.email "jenkins@test.com"
                        git config user.name "Jenkins CI"
                        git add k8s/deployment.yaml
                        git commit -m "release: update image to ${env.APP_VERSION} [skip ci]" || echo "No changes"
                        git push https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/${GITHUB_USERNAME}/test.git main
                    """
                }
                echo "✅ Manifests mis à jour sur GitHub"
            }
        }
        
        stage('Sync with ArgoCD') {
            steps {
                echo "🔄 Synchronisation avec ArgoCD..."
                sh """
                    argocd app sync test --grpc-web
                """
                echo "✅ Synchronisation déclenchée"
            }
        }
    }
    
    post {
        success {
            script {
                def IP = sh(script: "curl -s ifconfig.me", returnStdout: true).trim()
                echo """
                ════════════════════════════════════════════════════════════════
                ✅ PIPELINE RÉUSSI !
                ════════════════════════════════════════════════════════════════
                
                🐳 Image: ${IMAGE_NAME}:latest
                🌐 Application: http://${IP}:30082
                
                ════════════════════════════════════════════════════════════════
                """
            }
        }
        failure {
            echo "❌ PIPELINE ÉCHOUÉ - Vérifiez les logs"
        }
    }
}