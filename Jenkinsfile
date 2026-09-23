pipeline {
    agent any

    parameters {
        // Override this at "Build with Parameters" once you know your EKS Ingress
        // address (ALB hostname, or your own domain if you set one up).
        // Leave the default for local/Minikube builds.
        string(name: 'INGRESS_HOST', defaultValue: 'http://streamingapp.local', description: 'Base URL the frontend should call (Minikube host or EKS Ingress/ALB address)')
    }

    environment {
        AWS_ACCOUNT_ID = '740393042029'
        AWS_REGION     = 'us-east-1'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        // Unique, traceable tag per build — avoids the "latest never re-pulls" trap.
        IMAGE_TAG      = "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(7) ?: 'nogit'}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('AWS ECR Login') {
            steps {
                withCredentials([
                    string(credentialsId: 'HARSHRAJ_AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'HARSHRAJ_AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                }
            }
        }

        stage('Build & Push Auth Service') {
            steps {
                dir('StreamingApp') {
                    sh "docker build -t ${ECR_REGISTRY}/streaming-auth:${IMAGE_TAG} ./backend/authService"
                    sh "docker push ${ECR_REGISTRY}/streaming-auth:${IMAGE_TAG}"
                }
            }
        }

        stage('Build & Push Streaming Service') {
            steps {
                dir('StreamingApp') {
                    sh "docker build -t ${ECR_REGISTRY}/streaming-streaming:${IMAGE_TAG} -f ./backend/streamingService/Dockerfile ./backend"
                    sh "docker push ${ECR_REGISTRY}/streaming-streaming:${IMAGE_TAG}"
                }
            }
        }

        stage('Build & Push Admin Service') {
            steps {
                dir('StreamingApp') {
                    sh "docker build -t ${ECR_REGISTRY}/streaming-admin:${IMAGE_TAG} -f ./backend/adminService/Dockerfile ./backend"
                    sh "docker push ${ECR_REGISTRY}/streaming-admin:${IMAGE_TAG}"
                }
            }
        }

        stage('Build & Push Chat Service') {
            steps {
                dir('StreamingApp') {
                    sh "docker build -t ${ECR_REGISTRY}/streaming-chat:${IMAGE_TAG} -f ./backend/chatService/Dockerfile ./backend"
                    sh "docker push ${ECR_REGISTRY}/streaming-chat:${IMAGE_TAG}"
                }
            }
        }

        stage('Build & Push Frontend') {
            steps {
                dir('StreamingApp') {
                    sh """
                        docker build \
                          --build-arg REACT_APP_AUTH_API_URL=${params.INGRESS_HOST}/api/auth \
                          --build-arg REACT_APP_STREAMING_API_URL=${params.INGRESS_HOST}/api \
                          --build-arg REACT_APP_STREAMING_PUBLIC_URL=${params.INGRESS_HOST}/api/streaming \
                          --build-arg REACT_APP_ADMIN_API_URL=${params.INGRESS_HOST}/api/admin \
                          --build-arg REACT_APP_CHAT_API_URL=${params.INGRESS_HOST}/api/chat \
                          --build-arg REACT_APP_CHAT_SOCKET_URL=${params.INGRESS_HOST} \
                          -t ${ECR_REGISTRY}/streaming-frontend:${IMAGE_TAG} ./frontend
                    """
                    sh "docker push ${ECR_REGISTRY}/streaming-frontend:${IMAGE_TAG}"
                }
            }
        }

        stage('Print Image Tags') {
            steps {
                echo "Built and pushed all 5 images with tag: ${IMAGE_TAG}"
                echo "Update your Helm values.yaml (or run 'helm upgrade --set services.<name>.tag=${IMAGE_TAG}') to deploy these."
            }
        }
    }
}