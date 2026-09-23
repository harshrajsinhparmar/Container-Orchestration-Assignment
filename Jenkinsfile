pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '740393042029'
        AWS_REGION     = 'us-east-1'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('AWS ECR Login & Build') {
            steps {
                withCredentials([
                    string(credentialsId: 'HARSHRAJ_AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'HARSHRAJ_AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                    sh "docker build -t ${ECR_REGISTRY}/streaming-auth:latest ./backend/authService"
                    sh "docker push ${ECR_REGISTRY}/streaming-auth:latest"
                    sh "docker build -t ${ECR_REGISTRY}/streaming-streaming:latest -f ./backend/streamingService/Dockerfile ./backend"
                    sh "docker push ${ECR_REGISTRY}/streaming-streaming:latest"
                    sh "docker build -t ${ECR_REGISTRY}/streaming-admin:latest -f ./backend/adminService/Dockerfile ./backend"
                    sh "docker push ${ECR_REGISTRY}/streaming-admin:latest"
                    sh "docker build -t ${ECR_REGISTRY}/streaming-chat:latest -f ./backend/chatService/Dockerfile ./backend"
                    sh "docker push ${ECR_REGISTRY}/streaming-chat:latest"
                    sh "docker build --build-arg REACT_APP_AUTH_API_URL=http://streamingapp.local/api/auth --build-arg REACT_APP_STREAMING_API_URL=http://streamingapp.local/api --build-arg REACT_APP_STREAMING_PUBLIC_URL=http://streamingapp.local/api/streaming --build-arg REACT_APP_ADMIN_API_URL=http://streamingapp.local/api/admin --build-arg REACT_APP_CHAT_API_URL=http://streamingapp.local/api/chat --build-arg REACT_APP_CHAT_SOCKET_URL=http://streamingapp.local -t ${ECR_REGISTRY}/streaming-frontend:latest ./frontend"
                    sh "docker push ${ECR_REGISTRY}/streaming-frontend:latest"
                }
            }
        }
    }
}