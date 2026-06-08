pipeline {
    agent any

    environment {
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

        stage('Install & Build Application') {
            steps {
                // This command spins up a container, mounts your current workspace, 
                // installs dependencies, builds your app, and then cleanly deletes itself.
                sh '''
                    docker run --rm -v "$(pwd)":/app -w /app node:20-alpine sh -c "
                        npm ci && \
                        npm run lint --if-present && \
                        npm run build
                    "
                '''
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