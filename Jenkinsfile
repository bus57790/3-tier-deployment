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

        stage('SonarQube Quality Gate Enforcement') {
            steps {
                script {
                    // Increased timeout duration from 5 to 15 minutes
                    timeout(time: 15, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted: SonarQube Quality Gate failed with status ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Dependency & Security Vulnerability Scan') {
            steps {
                // Non-blocking report first, followed by strict blocking gate
                sh 'trivy fs --severity LOW,MEDIUM,HIGH,CRITICAL .'
                sh 'trivy fs --exit-code 1 --severity HIGH,CRITICAL .'
            }
        }

        stage('Build & Container Security Scan') {
            steps {
                script {
                    def envTag = params.ENVIRONMENT.toLowerCase()
                    
                    // Build Frontend & Backend Images
                    sh "docker build -t ${HARBOR_HOST}/${HARBOR_PROJECT}/frontend:${envTag}-${params.IMAGE_TAG} ./frontend"
                    sh "docker build -t ${HARBOR_HOST}/${HARBOR_PROJECT}/backend:${envTag}-${params.IMAGE_TAG} ./backend"

                    // Scan built backend image with Trivy (Fails build on HIGH or CRITICAL)
                    sh "trivy image --severity LOW,MEDIUM,HIGH,CRITICAL ${HARBOR_HOST}/${HARBOR_PROJECT}/backend:${envTag}-${params.IMAGE_TAG}"
                    sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${HARBOR_HOST}/${HARBOR_PROJECT}/backend:${envTag}-${params.IMAGE_TAG}"
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
                    
                    // Copy compose files to target node
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
        failure {
            script {
                def failedStage = "Unknown Stage"
                try {
                    def stageList = currentBuild.rawBuild.getExecution().getBlocks()
                    for (block in stageList) {
                        if (block.getTimingInfo() != null && block.getError() != null) {
                            failedStage = block.getDisplayName()
                            break
                        }
                    }
                } catch (Exception e) {
                    failedStage = "Pipeline Execution Failure"
                }

                def alertTitle = "🚨 DevSecOps Pipeline Failure"
                def alertMessage = "Build *#${env.BUILD_NUMBER}* for *${env.JOB_NAME}* failed at stage: *${failedStage}*."

                // Extract Slack and Teams Webhook secrets on demand
                withCredentials([
                    string(credentialsId: 'slack-webhook-url', variable: 'SLACK_URL'),
                    string(credentialsId: 'teams-webhook-url', variable: 'TEAMS_URL')
                ]) {
                    sendSlackNotification(alertTitle, alertMessage, env.BUILD_URL, env.SLACK_URL)
                    sendTeamsNotification(alertTitle, alertMessage, env.BUILD_URL, failedStage, env.TEAMS_URL)
                }
            }
        }

        always {
            sh 'docker logout ${HARBOR_HOST} || true'
            cleanWs()
        }
    }
}

def sendSlackNotification(title, message, buildUrl, webhookUrl) {
    def payload = """{
        "attachments": [
            {
                "color": "#FF0000",
                "title": "${title}",
                "title_link": "${buildUrl}",
                "text": "${message}",
                "fields": [
                    { "title": "Environment", "value": "${params.ENVIRONMENT}", "short": true },
                    { "title": "Image Tag", "value": "${params.IMAGE_TAG}", "short": true }
                ]
            }
        ]
    }"""
    sh(script: "curl -s -X POST -H 'Content-type: application/json' --data '${payload.replaceAll('\n', '')}' ${webhookUrl} || true", returnStdout: true)
}

def sendTeamsNotification(title, message, buildUrl, failedStage, webhookUrl) {
    def payload = """{
        "type": "message",
        "attachments": [
            {
                "contentType": "application/vnd.microsoft.card.adaptive",
                "content": {
                    "\$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
                    "type": "AdaptiveCard",
                    "version": "1.2",
                    "body": [
                        { "type": "TextBlock", "text": "${title}", "weight": "Bolder", "size": "Large", "color": "Attention" },
                        { "type": "TextBlock", "text": "${message}", "wrap": true },
                        {
                            "type": "FactSet",
                            "facts": [
                                { "title": "Failed Stage:", "value": "${failedStage}" },
                                { "title": "Environment:", "value": "${params.ENVIRONMENT}" },
                                { "title": "Image Tag:", "value": "${params.IMAGE_TAG}" }
                            ]
                        }
                    ],
                    "actions": [
                        {
                            "type": "Action.OpenUrl",
                            "title": "View Jenkins Log",
                            "url": "${buildUrl}"
                        }
                    ]
                }
            }
        ]
    }"""
    sh(script: "curl -s -X POST -H 'Content-type: application/json' --data '${payload.replaceAll('\n', '')}' ${webhookUrl} || true", returnStdout: true)
}
