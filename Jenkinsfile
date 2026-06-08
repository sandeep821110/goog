pipeline {
    agent any

    environment {
        // Your Docker Hub details
        DOCKER_IMAGE  = 'sandeep821110/goog' 
        DOCKER_TAG    = "${env.BUILD_NUMBER}"
        K8S_NAMESPACE = 'default'
        
        // Jenkins Credentials IDs
        DOCKER_CREDS_ID = 'docker-hub-credentials'
        KUBE_CONFIG_ID  = 'kube-config-credentials'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push Image inside K8s') {
            steps {
                // This uses your kube-config to launch a temporary builder pod inside Kubernetes
                configFileProvider([configFile(fileId: "${KUBE_CONFIG_ID}", variable: 'KUBECONFIG')]) {
                    // Create a secret for Docker Hub inside Kubernetes so Kaniko can push the image
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS_ID}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh """
                            kubectl create secret docker-registry regcred \
                                --docker-server=https://index.docker.io/v1/ \
                                --docker-username=${DOCKER_USERNAME} \
                                --docker-password=${DOCKER_PASSWORD} \
                                --namespace=${K8S_NAMESPACE} \
                                --dry-run=client -o yaml | kubectl apply -f -
                        """
                    }

                    // Run Kaniko pod to clone your repo, build the image, and push it to Docker Hub
                    sh """
                        kubectl delete pod kaniko-builder --namespace=${K8S_NAMESPACE} --ignore-not-found=true
                        
                        kubectl run kaniko-builder \
                            --namespace=${K8S_NAMESPACE} \
                            --restart=Never \
                            --image=gcr.io/kaniko-project/executor:latest \
                            -- --context=git://github.com/sandeep821110/goog.git \
                               --destination=${DOCKER_IMAGE}:${DOCKER_TAG} \
                               --destination=${DOCKER_IMAGE}:latest
                        
                        echo "Waiting for Kaniko build to finish..."
                        kubectl wait --namespace=${K8S_NAMESPACE} --for=condition=Ready pod/kaniko-builder --timeout=60s || true
                        kubectl logs -f kaniko-builder --namespace=${K8S_NAMESPACE}
                    """
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
        always {
            // Clean up the builder pod after execution
            configFileProvider([configFile(fileId: "${KUBE_CONFIG_ID}", variable: 'KUBECONFIG')]) {
                sh "kubectl delete pod kaniko-builder --namespace=${K8S_NAMESPACE} --ignore-not-found=true"
            }
        }
        failure {
            echo 'Pipeline failed!'
        }
        success {
            echo 'Pipeline succeeded!'
        }
    }
}