pipeline {
    agent any

    environment {
        // Change this to your actual Docker Hub username or Registry URL (e.g., 'yourusername/goog')
        DOCKER_IMAGE  = 'sandeep821110/goog' 
        DOCKER_TAG    = "${env.BUILD_NUMBER}"
        K8S_NAMESPACE = 'default'
        
        // Credentials IDs configured inside Jenkins
        DOCKER_CREDS_ID = 'docker-hub-credentials'
        KUBE_CONFIG_ID  = 'kube-config-credentials'
    }

    tools {
        // 1. Fixes the 'npm: not found' error. 
        // Ensure you have installed the NodeJS plugin and named it 'NodeJS_20' in Global Tool Configuration.
        nodejs 'NodeJS_20' 
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                // 'npm ci' is perfect for CI environments
                sh 'npm ci'
            }
        }

        stage('Lint') {
            steps {
                // Added a catch or script check in case lint isn't configured in package.json
                sh 'npm run lint --if-present'
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
            }
        }

        stage('Push Docker Image') {
            steps {
                // 2. Wrap push commands in a credentials block so Jenkins can log in to your registry
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS_ID}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push ${DOCKER_IMAGE}:latest"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // 3. Securely pass the Kubernetes config file so kubectl can communicate with your cluster
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