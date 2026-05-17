pipeline {
    agent any

    environment {
        GITHUB_REPO    = 'https://github.com/HusnainNadeem/fileforge-docker-aws1212.git'
        BRANCH         = 'main'
        REMOTE_SERVER  = '3.145.179.209'
        REMOTE_USER    = 'ec2-user'
        APP_DIR        = '/home/ec2-user/fileforge'
    }

    stages {

        // ─── STAGE 1 ────────────────────────────────────────────────
        stage('Stop Running Containers') {
            steps {
                echo '🛑 Stopping all running containers...'
                sh '''
                    docker ps -q -f name=frontend  | xargs -r docker stop
                    docker ps -aq -f name=frontend | xargs -r docker rm

                    docker ps -q -f name=backend   | xargs -r docker stop
                    docker ps -aq -f name=backend  | xargs -r docker rm

                    docker ps -q -f name=mongo     | xargs -r docker stop
                    docker ps -aq -f name=mongo    | xargs -r docker rm

                    docker ps -q -f name=nginx     | xargs -r docker stop
                    docker ps -aq -f name=nginx    | xargs -r docker rm

                    echo "✅ All containers stopped and removed."
                '''
            }
        }

        // ─── STAGE 2 ────────────────────────────────────────────────
        stage('Pull Code from GitHub') {
            steps {
                echo '📥 Pulling latest code from GitHub...'
                git credentialsId: 'github-credentials',
                    url: "${GITHUB_REPO}",
                    branch: "${BRANCH}"
                echo '✅ Code pulled successfully.'
            }
        }

        // ─── STAGE 3 ────────────────────────────────────────────────
        stage('Build Docker Images') {
            parallel {

                stage('Build Frontend Image') {
                    steps {
                        echo '🔨 Building Frontend image...'
                        sh '''
                            docker build -t frontend:latest ./frontend
                            echo "✅ Frontend image built."
                        '''
                    }
                }

                stage('Build Backend Image') {
                    steps {
                        echo '🔨 Building Backend image...'
                        sh '''
                            docker build -t backend:latest ./backend
                            echo "✅ Backend image built."
                        '''
                    }
                }

            }
        }

        // ─── STAGE 4 ────────────────────────────────────────────────
        stage('Run Containers via Docker Compose') {
            steps {
                echo '🚀 Starting all containers with docker-compose...'
                sh '''
                    docker-compose -f docker-compose.yaml down --remove-orphans || true
                    docker-compose -f docker-compose.yaml up -d
                    echo "✅ All containers started successfully."
                    docker ps
                '''
            }
        }

        // ─── STAGE 5 ────────────────────────────────────────────────
        stage('Deploy to Server 3.145.179.209') {
            steps {
                echo '🌍 Deploying to remote server 3.145.179.209...'
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'aws-server-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh '''
                        # ── Create app directory on remote server ──
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no \
                            ${REMOTE_USER}@${REMOTE_SERVER} \
                            "mkdir -p ${APP_DIR}/frontend \
                                      ${APP_DIR}/backend \
                                      ${APP_DIR}/nginx"

                        # ── Copy all project files to remote server ──
                        scp -i $SSH_KEY -o StrictHostKeyChecking=no \
                            docker-compose.yaml \
                            ${REMOTE_USER}@${REMOTE_SERVER}:${APP_DIR}/

                        scp -i $SSH_KEY -o StrictHostKeyChecking=no \
                            nginx/nginx.conf \
                            ${REMOTE_USER}@${REMOTE_SERVER}:${APP_DIR}/nginx/

                        scp -i $SSH_KEY -o StrictHostKeyChecking=no -r \
                            frontend/ \
                            ${REMOTE_USER}@${REMOTE_SERVER}:${APP_DIR}/

                        scp -i $SSH_KEY -o StrictHostKeyChecking=no -r \
                            backend/ \
                            ${REMOTE_USER}@${REMOTE_SERVER}:${APP_DIR}/

                        # ── SSH into server and deploy ──
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no \
                            ${REMOTE_USER}@${REMOTE_SERVER} << 'EOF'

                            cd /home/ec2-user/fileforge

                            echo "🛑 Stopping old containers..."
                            docker-compose down --remove-orphans || true

                            echo "🔨 Building fresh images on server..."
                            docker build -t frontend:latest ./frontend
                            docker build -t backend:latest ./backend

                            echo "🚀 Starting all containers..."
                            docker-compose up -d

                            echo "✅ Deployment complete!"
                            docker ps
EOF
                    '''
                }
                echo '✅ Website deployed successfully!'
            }
        }

    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
            echo '🌐 Website Live → http://3.145.179.209'
        }
        failure {
            echo '❌ Pipeline failed! Check logs above.'
        }
    }
}
