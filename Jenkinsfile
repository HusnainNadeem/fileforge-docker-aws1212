pipeline {
    agent any

    environment {
        GITHUB_REPO = 'https://github.com/HusnainNadeem/fileforge-docker-aws1212.git'
        BRANCH = 'main'
        IMAGE_NAME = 'fileforge-app'
        CONTAINER_NAME = 'fileforge-container'
        REMOTE_SERVER = '3.145.179.209'
        REMOTE_USER = 'ubuntu'
    }

    stages {

        stage('Stop Running Containers') {
            steps {
                echo '🛑 Stopping running containers...'
                sh '''
                    docker ps -q -f name=${CONTAINER_NAME} | xargs -r docker stop
                    docker ps -aq -f name=${CONTAINER_NAME} | xargs -r docker rm
                    echo "✅ Containers stopped and removed."
                '''
            }
        }

        stage('Pull Code from GitHub') {
            steps {
                echo '📥 Pulling code from GitHub...'
                git credentialsId: 'github-credentials',
                    url: "${GITHUB_REPO}",
                    branch: "${BRANCH}"
                echo '✅ Code pulled successfully.'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '🔨 Building Docker image...'
                sh '''
                    docker build -t ${IMAGE_NAME}:latest .
                    echo "✅ Docker image built successfully."
                '''
            }
        }

        stage('Run New Container') {
            steps {
                echo '🚀 Starting new container...'
                sh '''
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p 80:80 \
                        --restart unless-stopped \
                        ${IMAGE_NAME}:latest
                    echo "✅ Container started successfully."
                '''
            }
        }

        stage('Deploy to Server 3.145.179.209') {
            steps {
                echo '🌍 Deploying website to remote server...'
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'aws-server-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh '''
                        # Copy docker-compose or project files to remote server
                        scp -i $SSH_KEY -o StrictHostKeyChecking=no \
                            docker-compose.yaml \
                            ${REMOTE_USER}@${REMOTE_SERVER}:/home/${REMOTE_USER}/app/

                        # SSH into remote server and deploy
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no \
                            ${REMOTE_USER}@${REMOTE_SERVER} << 'EOF'

                            echo "🛑 Stopping old containers..."
                            cd /home/ec2-user/app
                            docker ps -q -f name=fileforge-container | xargs -r docker stop
                            docker ps -aq -f name=fileforge-container | xargs -r docker rm

                            echo "🔨 Pulling latest Docker image..."
                            docker pull fileforge-app:latest || true

                            echo "🚀 Starting new container..."
                            docker run -d \
                                --name fileforge-container \
                                -p 80:80 \
                                --restart unless-stopped \
                                fileforge-app:latest

                            echo "✅ Deployment successful!"
                            docker ps
EOF
                    '''
                }
                echo '✅ Website deployed successfully on 3.145.179.209'
            }
        }

    }

    post {
        success {
            echo '✅ Pipeline completed! Website is live at http://3.145.179.209'
        }
        failure {
            echo '❌ Pipeline failed! Check logs above.'
        }
    }
}
