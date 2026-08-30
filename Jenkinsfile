pipeline {
    agent any

    environment {
        HARBOR_HOST    = '192.168.1.184'
        HARBOR_PROJECT = '3tier'
        HARBOR_CREDS   = credentials('harbor-credentials')
        TARGET_NODE    = '192.168.1.183'
        TARGET_USER    = 'deployer'
        SONAR_HOST     = 'http://192.168.1.184:9000'
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['DEV', 'UAT', 'PROD'], description: 'Select deployment target environment')
        string(name: 'IMAGE_TAG', defaultValue: 'v1.0.0', description: 'Container image tag')
    }

    stages {
        stage('Checkout Source') {
            steps {
                git branch: "${params.ENVIRONMENT == 'PROD' ? 'main' : params.ENVIRONMENT.toLowerCase()}",
                    url: "https://github.com/bus57790/3-tier-deployment.git"
            }
        }

        stage('Backend Build & Unit Test') {
            steps {
                dir('backend') {
                    sh 'mvn clean test'
                }
            }
        }

        stage('SonarQube Security Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    dir('backend') {
                        sh """
                            mvn sonar:sonar \
                            -Dsonar.projectKey=3tier-backend-${params.ENVIRONMENT.toLowerCase()} \
                            -Dsonar.host.url=${SONAR_HOST}
                        """
                    }
                }
            }
        }

        stage('Dependency & Security Vulnerability Scan') {
            steps {
                sh 'trivy fs --security-checks vuln,config .'
            }
        }

        stage('Build & Container Security Scan') {
            steps {
                script {
                    def envTag = params.ENVIRONMENT.toLowerCase()
                    
                    // Build Frontend & Backend Images
                    sh "docker build -t ${HARBOR_HOST}/${HARBOR_PROJECT}/frontend:${envTag}-${params.IMAGE_TAG} ./frontend"
                    sh "docker build -t ${HARBOR_HOST}/${HARBOR_PROJECT}/backend:${envTag}-${params.IMAGE_TAG} ./backend"

                    // Scan built images with Trivy
                    sh "trivy image ${HARBOR_HOST}/${HARBOR_PROJECT}/backend:${envTag}-${params.IMAGE_TAG}"
                }
            }
        }

        stage('Push Artifacts to Harbor') {
            steps {
                script {
                    def envTag = params.ENVIRONMENT.toLowerCase()
                    sh "echo ${HARBOR_CREDS_PSW} | docker login ${HARBOR_HOST} -u ${HARBOR_CREDS_USR} --password-stdin"
                    
                    sh "docker push ${HARBOR_HOST}/${HARBOR_PROJECT}/frontend:${envTag}-${params.IMAGE_TAG}"
                    sh "docker push ${HARBOR_HOST}/${HARBOR_PROJECT}/backend:${envTag}-${params.IMAGE_TAG}"
                    
                    // Tag as latest for environment docker-compose references
                    sh "docker tag ${HARBOR_HOST}/${HARBOR_PROJECT}/frontend:${envTag}-${params.IMAGE_TAG} ${HARBOR_HOST}/${HARBOR_PROJECT}/frontend:${envTag}-latest"
                    sh "docker tag ${HARBOR_HOST}/${HARBOR_PROJECT}/backend:${envTag}-${params.IMAGE_TAG} ${HARBOR_HOST}/${HARBOR_PROJECT}/backend:${envTag}-latest"
                    
                    sh "docker push ${HARBOR_HOST}/${HARBOR_PROJECT}/frontend:${envTag}-latest"
                    sh "docker push ${HARBOR_HOST}/${HARBOR_PROJECT}/backend:${envTag}-latest"
                }
            }
        }

        stage('Deploy to DEV/UAT Target Server (192.168.1.183)') {
            when {
                expression { params.ENVIRONMENT == 'DEV' || params.ENVIRONMENT == 'UAT' }
            }
            steps {
                script {
                    def envLower = params.ENVIRONMENT.toLowerCase()
                    
                    // Copy compose files to 192.168.1.183
                    sh "ssh ${TARGET_USER}@${TARGET_NODE} 'mkdir -p ~/app/${envLower}'"
                    sh "scp -r deployments/${envLower}/* ${TARGET_USER}@${TARGET_NODE}:~/app/${envLower}/"
                    sh "scp -r database ${TARGET_USER}@${TARGET_NODE}:~/app/"

                    // Execute Remote Docker Compose Deployment
                    sh """
                        ssh ${TARGET_USER}@${TARGET_NODE} '
                            cd ~/app/${envLower} && \
                            docker compose pull && \
                            docker compose up -d --remove-orphans
                        '
                    """
                }
            }
        }

        stage('GitOps Trigger for PROD (ArgoCD / Minikube)') {
            when {
                expression { params.ENVIRONMENT == 'PROD' }
            }
            steps {
                script {
                    // Update Kubernetes image tag in Git repository to trigger ArgoCD sync on minikube
                    sh """
                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@192.168.1.184"
                        sed -i 's|image: 192.168.1.184/3tier/backend:.*|image: 192.168.1.184/3tier/backend:prod-${params.IMAGE_TAG}|g' deployments/prod/k8s/02-backend.yaml
                        sed -i 's|image: 192.168.1.184/3tier/frontend:.*|image: 192.168.1.184/3tier/frontend:prod-${params.IMAGE_TAG}|g' deployments/prod/k8s/03-frontend.yaml
                        git add deployments/prod/k8s/
                        git commit -m "Update PROD image tag to ${params.IMAGE_TAG} [skip ci]"
                        git push origin main
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout ${HARBOR_HOST}'
            cleanWs()
        }
    }
}
