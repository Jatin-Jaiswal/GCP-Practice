pipeline {
    agent any

    environment {
        // Define common variables for easy management
        PROJECT_ID = 'iron-handler-471307-u7'
        GCR_REPO = 'gcp-practice-images'
        GCR_PATH_CLIENT = "us-central1-docker.pkg.dev/${PROJECT_ID}/${GCR_REPO}/client"
        GCR_PATH_SERVER = "us-central1-docker.pkg.dev/${PROJECT_ID}/${GCR_REPO}/server"
        CLUSTER_NAME = 'jenkins-cluster'
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
                // Using the Secret File credential and gcloud commands
                withCredentials([file(credentialsId: 'jenkins-gke-sa', variable: 'GCP_KEY_FILE')]) {
                    sh '''
                    echo "Activating Google Cloud Service Account..."
                    gcloud auth activate-service-account --key-file=${GCP_KEY_FILE}
                    gcloud config set project ${PROJECT_ID}

                    echo "Configuring Docker to use gcloud as a credential helper..."
                    gcloud auth configure-docker us-central1-docker.pkg.dev -q

                    echo "Pushing images with tag ${BUILD_NUMBER}..."
                    docker push ${GCR_PATH_SERVER}:${BUILD_NUMBER}
                    docker push ${GCR_PATH_CLIENT}:${BUILD_NUMBER}
                    
                    echo "Tagging and pushing 'latest' versions..."
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
                withCredentials([file(credentialsId: 'jenkins-gke-sa', variable: 'GCP_KEY_FILE')]) {
                    sh '''#!/bin/bash
                    echo "Activating Google Cloud Service Account for GKE..."
                    gcloud auth activate-service-account --key-file=${GCP_KEY_FILE}
                    
                    echo "Fetching GKE cluster credentials..."
                    gcloud container clusters get-credentials ${CLUSTER_NAME} --zone us-central1-a --project ${PROJECT_ID}

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
