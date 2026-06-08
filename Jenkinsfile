pipeline {
    agent any

    environment {
        // Docker registry repo target
        DOCKER_IMAGE  = 'sandeep821110/goog' 
        DOCKER_TAG    = "${env.BUILD_NUMBER}"
        K8S_NAMESPACE = 'default'
        
        // Credentials IDs inside Jenkins
        DOCKER_CREDS_ID = 'docker-hub-credentials'
        KUBE_CONFIG_ID  = 'kube-config-credentials'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // We run the application build steps inside a clean node container 
        stage('Build Frontend application') {
            agent {
                docker {
                    image 'node:20-alpine'
                    reuseNode true // Keeps your checked-out git workspace intact
                }
            }
            steps {
                // These run securely inside the Node 20 container container
                sh 'npm ci'
                sh 'npm run lint --if-present'
                sh 'npm run build'
            }
        }

        // We step back out to the native host agent to run docker commands
        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS_ID}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push ${DOCKER_IMAGE}:latest"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                configFileProvider([configFile(fileId: "${KUBE_CONFIG_ID}", variable: 'KUBECONFIG')]) {
                    sh """
                        kubectl set image deployment/goog \
                          goog=${DOCKER_IMAGE}:${DOCKER_TAG} \
                          --namespace=${K8S_NAMESPACE}
                    """
                }
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed!'
        }
        success {
            echo 'Pipeline succeeded!'
        }
    }
}