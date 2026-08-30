pipeline {
    agent any

    environment {
        HARBOR_HOST    = '192.168.1.184:9443'
        HARBOR_PROJECT = '3tier'
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
                        script {
                            def envKey = params.ENVIRONMENT.toLowerCase()
                            sh "mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=3tier-backend-${envKey} -Dsonar.host.url=${SONAR_HOST}"
                        }
                    }
                }
            }
        }

        stage('Build Container Images') {
            steps {
                script {
                    def envTag = params.ENVIRONMENT.toLowerCase()
                    
                    sh "docker build -t ${HARBOR_HOST}/${HARBOR_PROJECT}/frontend:${envTag}-${params.IMAGE_TAG} ./frontend"
                    sh "docker build -t ${HARBOR_HOST}/${HARBOR_PROJECT}/backend:${envTag}-${params.IMAGE_TAG} ./backend"
                }
            }
        }

        stage('Push Artifacts to Harbor') {
            steps {
                script {
                    def envTag = params.ENVIRONMENT.toLowerCase()
                    
                    withCredentials([usernamePassword(credentialsId: 'harbor-credentials', passwordVariable: 'HARBOR_PSW', usernameVariable: 'HARBOR_USR')]) {
                        sh 'echo "$HARBOR_PSW" | docker login ' + HARBOR_HOST + ' -u "$HARBOR_USR" --password-stdin'
                    }
                    
                    sh "docker push ${HARBOR_HOST}/${HARBOR_PROJECT}/frontend:${envTag}-${params.IMAGE_TAG}"
                    sh "docker push ${HARBOR_HOST}/${HARBOR_PROJECT}/backend:${envTag}-${params.IMAGE_TAG}"
                    
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
                    def sshFlags = "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
                    
                    withCredentials([usernamePassword(credentialsId: 'harbor-credentials', passwordVariable: 'HARBOR_PSW', usernameVariable: 'HARBOR_USR')]) {
                        sh "ssh ${sshFlags} ${TARGET_USER}@${TARGET_NODE} 'mkdir -p ~/app/${envLower}'"
                        sh "scp ${sshFlags} -r deployments/${envLower}/* ${TARGET_USER}@${TARGET_NODE}:~/app/${envLower}/"
                        sh "scp ${sshFlags} -r database ${TARGET_USER}@${TARGET_NODE}:~/app/"

                        // Pass credentials safely via SSH environment variables to prevent escape issues
                        sh """
                            ssh ${sshFlags} ${TARGET_USER}@${TARGET_NODE} "HARBOR_PSW='${HARBOR_PSW}' HARBOR_USR='${HARBOR_USR}' sh -c 'echo \\"\$HARBOR_PSW\\" | docker login ${HARBOR_HOST} -u \\"\$HARBOR_USR\\" --password-stdin' && \
                            cd ~/app/${envLower} && \
                            docker compose pull && \
                            docker compose up -d --remove-orphans"
                        """
                    }
                }
            }
        }

        stage('GitOps Trigger for PROD (ArgoCD / Minikube)') {
            when {
                expression { params.ENVIRONMENT == 'PROD' }
            }
            steps {
                script {
                    sh 'git config user.name "Jenkins CI"'
                    sh 'git config user.email "jenkins@192.168.1.184"'
                    sh "sed -i 's|image: 192.168.1.184:9443/3tier/backend:.*|image: 192.168.1.184:9443/3tier/backend:prod-${params.IMAGE_TAG}|g' deployments/prod/k8s/02-backend.yaml"
                    sh "sed -i 's|image: 192.168.1.184:9443/3tier/frontend:.*|image: 192.168.1.184:9443/3tier/frontend:prod-${params.IMAGE_TAG}|g' deployments/prod/k8s/03-frontend.yaml"
                    sh 'git add deployments/prod/k8s/'
                    sh "git commit -m 'Update PROD image tag to ${params.IMAGE_TAG} [skip ci]'"
                    sh 'git push origin main'
                }
            }
        }
    }

    post {
        failure {
            script {
                def alertTitle = "🚨 DevSecOps Pipeline Failure"
                def alertMessage = "Build *#${env.BUILD_NUMBER}* for *${env.JOB_NAME}* failed."

                withCredentials([
                    string(credentialsId: 'slack-webhook-url', variable: 'SLACK_URL')
                ]) {
                    sendSlackNotification(alertTitle, alertMessage, env.BUILD_URL)
                }
            }
        }

        always {
            script {
                def sshFlags = "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
                // Local build node logout
                sh "docker logout ${HARBOR_HOST} || true"
                // Target deployment node logout
                if (params.ENVIRONMENT == 'DEV' || params.ENVIRONMENT == 'UAT') {
                    sh "ssh ${sshFlags} ${TARGET_USER}@${TARGET_NODE} 'docker logout ${HARBOR_HOST}' || true"
                }
            }
            cleanWs()
        }
    }
}

def sendSlackNotification(title, message, buildUrl) {
    def jsonPayload = """{
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
    }""".replaceAll("\n", "").replaceAll("\\s+", " ")

    sh(script: "curl -s -X POST -H 'Content-type: application/json' --data '${jsonPayload}' \"\$SLACK_URL\" || true")
}
