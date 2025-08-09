pipeline {
    agent any

    environment {
        DOCKER_HUB_REPO = 'sharad9642/knowledgehub-app'
        K8S_CLUSTER_NAME = 'sharad-cluster'
        AWS_REGION = 'us-east-1'
        NAMESPACE = 'default'
        APP_NAME = 'knowledgehub'
    }

    stage('Checkout') {
    steps {
        echo 'Checking out source code...'
        git branch: 'main', url: 'https://github.com/Sharad-Parit6094/microservices-ingress-App.git'
    }
}

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                script {
                    def buildNumber = env.BUILD_NUMBER
                    def imageTag = "${DOCKER_HUB_REPO}:${buildNumber}"
                    def latestTag = "${DOCKER_HUB_REPO}:latest"

                    sh "docker build -t ${imageTag} ."
                    sh "docker tag ${imageTag} ${latestTag}"

                    env.IMAGE_TAG = buildNumber
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo 'Pushing Docker image to DockerHub...'
                script {
                    sh "docker logout || true" // ensure clean login
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin"
                        sh "docker push ${DOCKER_HUB_REPO}:${env.IMAGE_TAG}"
                        sh "docker push ${DOCKER_HUB_REPO}:latest"
                    }
                }
            }
        }

        stage('Configure AWS and Kubectl') {
            steps {
                echo 'Configuring AWS CLI and kubectl...'
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                        sh """
                            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                            aws configure set region ${AWS_REGION}
                        """
                        sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${K8S_CLUSTER_NAME}"
                        sh "kubectl config current-context"
                        sh "kubectl get nodes"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying application to Kubernetes...'
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                        sh "kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"
                        sh "sed -i 's|${DOCKER_HUB_REPO}:latest|${DOCKER_HUB_REPO}:${env.IMAGE_TAG}|g' k8s/deployment.yaml"
                        sh "kubectl apply -f k8s/deployment.yaml"
                        sh "kubectl rollout status deployment/${APP_NAME}-deployment --timeout=300s"
                        sh "kubectl get pods -l app=${APP_NAME}"
                        sh "kubectl get svc ${APP_NAME}-service"
                    }
                }
            }
        }

        stage('Deploy Ingress') {
            steps {
                echo 'Deploying Ingress resource...'
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                        sh "kubectl apply -f k8s/ingress.yaml"
                        sleep(10)
                        sh "kubectl get ingress ${APP_NAME}-ingress"
                        sh "kubectl describe ingress ${APP_NAME}-ingress"
                    }
                }
            }
        }

        stage('Get Ingress URL') {
            steps {
                echo 'Getting Ingress URL...'
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                        timeout(time: 10, unit: 'MINUTES') {
                            waitUntil {
                                def result = sh(
                                    script: "kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{.status.loadBalancer.ingress[0].ip}'",
                                    returnStdout: true
                                ).trim()
                                if (result) {
                                    env.INGRESS_URL = "http://${result}"
                                    echo "Ingress URL: ${env.INGRESS_URL}"
                                    return true
                                }
                                return false
                            }
                        }

                        echo "========================================="
                        echo "DEPLOYMENT SUCCESSFUL!"
                        echo "========================================="
                        echo "Application URL: ${env.INGRESS_URL}"
                        echo ""
                        echo "Available Paths:"
                        echo "- Home Page: ${env.INGRESS_URL}/"
                        echo "- About Page: ${env.INGRESS_URL}/about"
                        echo "- Services Page: ${env.INGRESS_URL}/services"
                        echo "- Contact Page: ${env.INGRESS_URL}/contact"
                        echo "========================================="

                        retry(5) {
                            sh "curl -I ${env.INGRESS_URL}/ || sleep 10"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up Docker images...'
            sh "docker rmi ${DOCKER_HUB_REPO}:${env.IMAGE_TAG} || true"
            sh "docker rmi ${DOCKER_HUB_REPO}:latest || true"
        }

        success {
            echo 'Pipeline completed successfully!'
            echo "Access your application at: ${env.INGRESS_URL}"
        }

        failure {
            echo 'Pipeline failed! Please check the logs.'
        }
    }
}
