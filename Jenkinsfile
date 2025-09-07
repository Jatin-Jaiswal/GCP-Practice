pipeline {
    agent any

    // Ensure kubectl is available on the Jenkins agent
    tools {
        // Change this if you have a different version of kubectl configured
        // in Jenkins Global Tool Configuration
        kubernetes 'kubectl'
    }

    environment {
        // Define common variables for easy management
        PROJECT_ID = 'iron-handler-471307-u7'
        GCR_REPO = 'gcp-practice-images'
        GCR_PATH_CLIENT = "us-central1-docker.pkg.dev/${PROJECT_ID}/${GCR_REPO}/client"
        GCR_PATH_SERVER = "us-central1-docker.pkg.dev/${PROJECT_ID}/${GCR_REPO}/server"
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'google-chat-url', variable: 'GOOGLE_CHAT_URL')]) {
                        googlechatnotification url: "${GOOGLE_CHAT_URL}",
                        message: "🔔 Build #${env.BUILD_NUMBER} for ${env.JOB_NAME} started."
                    }
                }
                // Use HTTPS for cloning with the github-pat credential
                git branch: 'main', credentialsId: 'github-pat', url: 'https://github.com/Jatin-Jaiswal/GCP-Practice.git'
            }
        }
        
        stage('Docker Build') {
            steps {
                 script {
                    sh "docker build -t ${GCR_PATH_SERVER}:${env.BUILD_NUMBER} ./server"
                    sh "docker build -t ${GCR_PATH_CLIENT}:${env.BUILD_NUMBER} ./client"
                }
            }
        }

        stage('Push Docker Images to GCR') {
            steps {
                // Use the Google Service Account credentials to authenticate with GCR
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_SA_KEY')]) {
                    sh '''
                    # Login to GCR using the service account key
                    cat "${GCP_SA_KEY}" | docker login -u _json_key_base64 --password-stdin https://us-central1-docker.pkg.dev
                    
                    # Push the backend and frontend images with the build number tag
                    docker push ${GCR_PATH_SERVER}:${BUILD_NUMBER}
                    docker push ${GCR_PATH_CLIENT}:${BUILD_NUMBER}
                    
                    # Also push a 'latest' tag for convenience
                    docker tag ${GCR_PATH_SERVER}:${BUILD_NUMBER} ${GCR_PATH_SERVER}:latest
                    docker tag ${GCR_PATH_CLIENT}:${BUILD_NUMBER} ${GCR_PATH_CLIENT}:latest
                    docker push ${GCR_PATH_SERVER}:latest
                    docker push ${GCR_PATH_CLIENT}:latest
                    '''
                }
            }
        }
        
           sstage('Deploy to Kubernetes') {
            steps {
               script {
                    sh '''#!/bin/bash
                    echo "Applying Kubernetes deployments and services..."
                    
                    # Apply all YAML files
                    kubectl apply -f ./server/server-deployment.yaml
                    kubectl apply -f ./server/server-service.yaml
                    kubectl apply -f ./client/client-deployment.yaml
                    kubectl apply -f ./client/client-service.yaml

                    # Rolling restart to apply changes
                    kubectl rollout restart deployment/client-deployment
                    kubectl rollout restart deployment/server-deployment
                    '''
                }
            }
        }
    }
    
    post {
        always {
            script {
                def log = currentBuild.rawBuild.getLog(100).join('\n')
                writeFile file: 'console.log', text: log
            }
        }
         success {
            script {
                def log = readFile('console.log')
                withCredentials([string(credentialsId: 'google-chat-url', variable: 'GOOGLE_CHAT_URL')]) {
                    def message = "✅ ${env.JOB_NAME} : Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}: Check output at ${env.BUILD_URL}\n\nLog:\n${log.take(4000)}"
                    echo "Google Chat URL: ${GOOGLE_CHAT_URL}"
                    googlechatnotification url: "${GOOGLE_CHAT_URL}",
                    message: message
                }
                cleanWs()
            }
        }
        failure {
            script {
                def log = readFile('console.log')
                withCredentials([string(credentialsId: 'google-chat-url', variable: 'GOOGLE_CHAT_URL')]) {
                    def message = "❌ ${env.JOB_NAME} : Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}: Check output at ${env.BUILD_URL}\n\nLog:\n${log.take(4000)}"
                    echo "Google Chat URL: ${GOOGLE_CHAT_URL}"
                    googlechatnotification url: "${GOOGLE_CHAT_URL}",
                    message: message
                }
                cleanWs()
            }
        }
    }
}
