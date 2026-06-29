pipeline {
    agent any
    
    triggers {
        pollSCM('*/2 * * * *') // Automatically scans your prod branch every 2 minutes
    }

    environment {
        DOCKER_USER    = 'abhishekdocker03'
        GITHUB_USER    = 'AbhishekChoudhary23'
        GITOPS_REPO    = 'github.com/AbhishekChoudhary23/DevOps-Observability-k8s-gitops-manifest.git'
        BUILD_TAG      = "prod-${BUILD_NUMBER}" // Creates a unique tag, e.g., prod-1, prod-2
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'prod', url: 'https://github.com/AbhishekChoudhary23/DevOps-Observability-app.git'
            }
        }

        stage('Build & Push Production Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_PASS')]) {
                    sh "echo \$REGISTRY_PASS | docker login -u \$REGISTRY_USER --password-stdin"
                    
                    // Build and Push Backend
                    sh "docker build -t ${DOCKER_USER}/gitops-backend:${BUILD_TAG} ./backend"
                    sh "docker push ${DOCKER_USER}/gitops-backend:${BUILD_TAG}"
                    
                    // Build and Push Frontend
                    sh "docker build -t ${DOCKER_USER}/gitops-frontend:${BUILD_TAG} ./frontend"
                    sh "docker push ${DOCKER_USER}/gitops-frontend:${BUILD_TAG}"
                }
            }
        }

        stage('Update GitOps Manifests') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-gitops-creds', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    sh """
                        # Clone the GitOps deployment repo clean
                        rm -rf DevOps-Observability-k8s-gitops-manifest
                        git clone https://\$GH_TOKEN@${GITOPS_REPO}
                        cd DevOps-Observability-k8s-gitops-manifest
                        
                        # We will make sure the apps directory exists
                        mkdir -p apps
                        
                        # Dynamically write/overwrite the deployment tags using a standard bash stream edit
                        # (Don't worry, we will create the template manifests in the next step!)
                        sed -i 's|image: .*'/gitops-backend:.*|image: ${DOCKER_USER}/gitops-backend:${BUILD_TAG}|g' apps/backend-deployment.yaml || true
                        sed -i 's|image: .*'/gitops-frontend:.*|image: ${DOCKER_USER}/gitops-frontend:${BUILD_TAG}|g' apps/frontend-deployment.yaml || true
                        
                        # Commit the change back to the GitOps repo
                        git config user.name "Jenkins CI/CD"
                        git config user.email "jenkins@local.cluster"
                        git add .
                        git commit -m "image update: bump compilation tags to ${BUILD_TAG} [skip ci]" || echo "No changes to commit"
                        git push origin main
                    """
                }
            }
        }
    }
}