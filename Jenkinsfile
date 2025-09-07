pipeline {
    agent any

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
                // Now using the correct method for the Google Service Account credential type
                withCredentials([googleServiceAccount(credentialsId: 'jenkins-gke-sa', variable: 'GCP_SA_KEY')]) {
                    sh '''
                    # Login to GCR using the service account key
                    docker login -u _json_key_base64 --password-stdin https://us-central1-docker.pkg.dev <<< "${GCP_SA_KEY}"
                    
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
    
        stage('Deploy to Kubernetes') {
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
}
