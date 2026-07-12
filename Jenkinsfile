pipeline {
    agent any
    environment {
        FRONTEND_IMAGE = "streaming-app-frontend"
        AUTH_IMAGE = "streaming-app-auth"
        STREAMING_IMAGE = "streaming-app-streaming"
        ADMIN_IMAGE = "streaming-app-admin"
        CHAT_IMAGE = "streaming-app-chat"
    }
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }
        
        stage('AWS ECR Login & Push') {
            steps {
                // Bind AWS Credentials, Account ID, and Default Region from Jenkins Credentials vault
                withCredentials([
                    string(credentialsId: 'HV-B16A-TB-AWS-ACCOUNT-ID', variable: 'AWS_ACCOUNT_ID'),
                    string(credentialsId: 'HV-B16A-TB-AWS-DEFAULT-REGION', variable: 'AWS_DEFAULT_REGION'),
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'HV-B16A-TB-AWS-CREDENTIALS']
                ]) {
                    script {
                        def ecrRegistry = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
                        
                        // Log in to AWS ECR
                        sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ecrRegistry}"
                        
                        // 1. Build and Push Auth Service (Context: backend/authService)
                        sh "docker build -t ${AUTH_IMAGE}:${BUILD_NUMBER} -f backend/authService/Dockerfile backend/authService"
                        sh "docker tag ${AUTH_IMAGE}:${BUILD_NUMBER} ${ecrRegistry}/${AUTH_IMAGE}:${BUILD_NUMBER}"
                        sh "docker tag ${AUTH_IMAGE}:${BUILD_NUMBER} ${ecrRegistry}/${AUTH_IMAGE}:latest"
                        sh "docker push ${ecrRegistry}/${AUTH_IMAGE}:${BUILD_NUMBER}"
                        sh "docker push ${ecrRegistry}/${AUTH_IMAGE}:latest"
                        
                        // 2. Build and Push Streaming Service (Context: backend)
                        sh "docker build -t ${STREAMING_IMAGE}:${BUILD_NUMBER} -f backend/streamingService/Dockerfile backend"
                        sh "docker tag ${STREAMING_IMAGE}:${BUILD_NUMBER} ${ecrRegistry}/${STREAMING_IMAGE}:${BUILD_NUMBER}"
                        sh "docker tag ${STREAMING_IMAGE}:${BUILD_NUMBER} ${ecrRegistry}/${STREAMING_IMAGE}:latest"
                        sh "docker push ${ecrRegistry}/${STREAMING_IMAGE}:${BUILD_NUMBER}"
                        sh "docker push ${ecrRegistry}/${STREAMING_IMAGE}:latest"
                        
                        // 3. Build and Push Admin Service (Context: backend)
                        sh "docker build -t ${ADMIN_IMAGE}:${BUILD_NUMBER} -f backend/adminService/Dockerfile backend"
                        sh "docker tag ${ADMIN_IMAGE}:${BUILD_NUMBER} ${ecrRegistry}/${ADMIN_IMAGE}:${BUILD_NUMBER}"
                        sh "docker tag ${ADMIN_IMAGE}:${BUILD_NUMBER} ${ecrRegistry}/${ADMIN_IMAGE}:latest"
                        sh "docker push ${ecrRegistry}/${ADMIN_IMAGE}:${BUILD_NUMBER}"
                        sh "docker push ${ecrRegistry}/${ADMIN_IMAGE}:latest"
                        
                        // 4. Build and Push Chat Service (Context: backend)
                        sh "docker build -t ${CHAT_IMAGE}:${BUILD_NUMBER} -f backend/chatService/Dockerfile backend"
                        sh "docker tag ${CHAT_IMAGE}:${BUILD_NUMBER} ${ecrRegistry}/${CHAT_IMAGE}:${BUILD_NUMBER}"
                        sh "docker tag ${CHAT_IMAGE}:${BUILD_NUMBER} ${ecrRegistry}/${CHAT_IMAGE}:latest"
                        sh "docker push ${ecrRegistry}/${CHAT_IMAGE}:${BUILD_NUMBER}"
                        sh "docker push ${ecrRegistry}/${CHAT_IMAGE}:latest"
                        
                        // 5. Build and Push Frontend (Context: frontend, with relative proxy build variables)
                        sh "docker build -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} " +
                           "--build-arg REACT_APP_AUTH_API_URL=/api/auth " +
                           "--build-arg REACT_APP_STREAMING_API_URL=/api/streaming " +
                           "--build-arg REACT_APP_STREAMING_PUBLIC_URL=/api/streaming " +
                           "--build-arg REACT_APP_ADMIN_API_URL=/api/admin " +
                           "--build-arg REACT_APP_CHAT_API_URL=/api/chat " +
                           "--build-arg REACT_APP_CHAT_SOCKET_URL=/ " +
                           "-f frontend/Dockerfile frontend"
                        sh "docker tag ${FRONTEND_IMAGE}:${BUILD_NUMBER} ${ecrRegistry}/${FRONTEND_IMAGE}:${BUILD_NUMBER}"
                        sh "docker tag ${FRONTEND_IMAGE}:${BUILD_NUMBER} ${ecrRegistry}/${FRONTEND_IMAGE}:latest"
                        sh "docker push ${ecrRegistry}/${FRONTEND_IMAGE}:${BUILD_NUMBER}"
                        sh "docker push ${ecrRegistry}/${FRONTEND_IMAGE}:latest"
                    }
                }
            }
        }
    }
    post {
        always {
            // Clean up unused docker images to save disk space on Jenkins builder
            sh "docker image prune -f"
        }
    }
}