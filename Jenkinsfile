// Jenkinsfile
pipeline {
    agent any
    
    environment {
        // À MODIFIER AVEC VOS INFOS
        DOCKER_HUB_USERNAME = 'sasow'
        DOCKER_HUB_REPO = 'immo-app'
        IMAGE_NAME = "${DOCKER_HUB_USERNAME}/${DOCKER_HUB_REPO}"
        GITHUB_REPO = 'https://github.com/SalifAbdoulSow18/immo-app.git'
        K8S_NAMESPACE = 'immo-app'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "📦 Récupération du code depuis GitHub..."
                git branch: 'main', 
                    url: GITHUB_REPO,
                    credentialsId: 'github-credentials'
                echo "✅ Code récupéré - Commit: ${env.GIT_COMMIT}"
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo "🐳 Build de l'image Docker..."
                script {
                    // Build avec tag commit et latest
                    docker.build("${IMAGE_NAME}:${env.GIT_COMMIT}")
                    docker.tag("${IMAGE_NAME}:${env.GIT_COMMIT}", "${IMAGE_NAME}:latest")
                }
                echo "✅ Image buildée: ${IMAGE_NAME}:${env.GIT_COMMIT}"
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                echo "📤 Push de l'image vers Docker Hub..."
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
                        docker.image("${IMAGE_NAME}:${env.GIT_COMMIT}").push()
                        docker.image("${IMAGE_NAME}:latest").push()
                    }
                }
                echo "✅ Image poussée sur Docker Hub"
            }
        }
        
        stage('Update Kubernetes Manifests') {
            steps {
                echo "📝 Mise à jour des manifests Kubernetes..."
                script {
                    sh """
                        # Mettre à jour l'image dans le deployment
                        sed -i "s|image: .*|image: ${IMAGE_NAME}:${env.GIT_COMMIT}|" k8s/deployment.yaml
                        
                        # Commit et push sur GitHub
                        git config user.email "jenkins@immoapp.com"
                        git config user.name "Jenkins CI"
                        git add k8s/deployment.yaml
                        git commit -m "chore: update image to ${IMAGE_NAME}:${env.GIT_COMMIT} [skip ci]" || echo "No changes to commit"
                        git push https://${env.GITHUB_CREDENTIALS_USR}:${env.GITHUB_CREDENTIALS_PSW}@github.com/VOTRE_USERNAME_GITHUB/immo-app.git main
                    """
                }
                echo "✅ Manifests mis à jour sur GitHub"
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo "☸️ Déploiement sur Kubernetes..."
                script {
                    sh """
                        # Créer le namespace s'il n'existe pas
                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Appliquer les manifests
                        kubectl apply -f k8s/namespace.yaml
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        
                        # Attendre le déploiement
                        kubectl rollout status deployment/immo-app -n ${K8S_NAMESPACE} --timeout=120s
                    """
                }
                echo "✅ Déploiement terminé"
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
                
                📊 Détails :
                   - Commit : ${env.GIT_COMMIT}
                   - Image : ${IMAGE_NAME}:${env.GIT_COMMIT}
                   - Namespace : ${K8S_NAMESPACE}
                
                🌐 URLs :
                   - Application : http://${IP}:30080
                   - Jenkins : http://${IP}:8080
                   - ArgoCD : https://${IP}:30443
                
                ════════════════════════════════════════════════════════════════
                """
            }
        }
        failure {
            echo """
            ════════════════════════════════════════════════════════════════
            ❌ PIPELINE ÉCHOUÉ !
            ════════════════════════════════════════════════════════════════
            
            Vérifiez les logs ci-dessus pour plus de détails.
            ════════════════════════════════════════════════════════════════
            """
        }
    }
}