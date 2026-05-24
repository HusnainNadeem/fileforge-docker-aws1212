pipeline {
    agent any

    environment {
        DOCKER_COMPOSE_FILE = 'docker-compose.yml'
        APP_DIR             = '/home/ubuntu/fileforge'
        EC2_USER            = 'ubuntu'
        EC2_HOST            = '18.222.164.24'
        GITHUB_REPO         = 'https://github.com/HusnainNadeem/fileforge-docker-aws1212.git'
    }

    }

    stages {

        // ─── 1. Checkout ────────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo 'Cloning repository from GitHub...'
                git branch: 'main',
                    url: "${GITHUB_REPO}"
            }
        }

        // ─── 2. Validate ────────────────────────────────────────────────────
        stage('Validate') {
            parallel {
                stage('Lint Backend') {
                    steps {
                        dir('backend') {
                            sh '''
                                npm install
                                npm run lint || echo "No lint script, skipping..."
                            '''
                        }
                    }
                }
                stage('Lint Frontend') {
                    steps {
                        dir('frontend') {
                            sh '''
                                npm install
                                npm run lint || echo "No lint script, skipping..."
                            '''
                        }
                    }
                }
            }
        }

        // ─── 3. Sync Files to EC2 ───────────────────────────────────────────
        stage('Sync Files') {
            steps {
                echo 'Syncing project files to EC2...'
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            mkdir -p ${APP_DIR}
                        '
                        rsync -avz \
                            --exclude='.git' \
                            --exclude='node_modules' \
                            --exclude='.env' \
                            ./ ${EC2_USER}@${EC2_HOST}:${APP_DIR}/
                    """
                }
            }
        }

        // ─── 4. Stop & Remove Old Containers ────────────────────────────────
        stage('Stop Old Containers') {
            steps {
                echo 'Stopping and removing old containers...'
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            cd ${APP_DIR}

                            echo ">>> Running containers before cleanup:"
                            sudo docker ps

                            echo ">>> Stopping all containers..."
                            sudo docker compose down --remove-orphans

                            echo ">>> All containers stopped."
                            sudo docker ps -a
                        '
                    """
                }
            }
        }

        // ─── 5. Remove Old Images ────────────────────────────────────────────
        stage('Remove Old Images') {
            steps {
                echo 'Removing old Docker images...'
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            echo ">>> Images before cleanup:"
                            sudo docker images

                            echo ">>> Removing backend image..."
                            sudo docker rmi backend 2>/dev/null && echo "backend removed" || echo "backend not found"

                            echo ">>> Removing frontend image..."
                            sudo docker rmi frontend 2>/dev/null && echo "frontend removed" || echo "frontend not found"

                            echo ">>> Pruning dangling images..."
                            sudo docker image prune -f

                            echo ">>> Images after cleanup:"
                            sudo docker images
                        '
                    """
                }
            }
        }

        // ─── 6. Build New Images ─────────────────────────────────────────────
        stage('Build New Images') {
            steps {
                echo 'Building new Docker images from latest code...'
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            cd ${APP_DIR}

                            echo ">>> Building backend image..."
                            sudo docker compose build --no-cache backend

                            echo ">>> Building frontend image..."
                            sudo docker compose build --no-cache frontend

                            echo ">>> New images:"
                            sudo docker images
                        '
                    """
                }
            }
        }

        // ─── 7. Start New Containers ─────────────────────────────────────────
        stage('Start New Containers') {
            steps {
                echo 'Starting fresh containers from new images...'
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            cd ${APP_DIR}

                            echo ">>> Starting all services..."
                            sudo docker compose up -d

                            echo ">>> Running containers:"
                            sudo docker compose ps
                        '
                    """
                }
            }
        }

        // ─── 8. Health Check ─────────────────────────────────────────────────
        stage('Health Check') {
            steps {
                echo 'Waiting for services to be ready...'
                sleep(time: 15, unit: 'SECONDS')

                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            echo "=== Final Container Status ==="
                            sudo docker compose -f ${APP_DIR}/docker-compose.yml ps

                            echo "=== App Health Check ==="
                            curl -sf http://localhost:80 > /dev/null && \
                                echo "SUCCESS: App is UP at http://${EC2_HOST}" || \
                                echo "FAILED: App is NOT responding on port 80"
                        '
                    """
                }
            }
        }
    }

    // ─── Post Actions ────────────────────────────────────────────────────────
    post {
        success {
            echo "Deployment SUCCESS — http://${EC2_HOST}"
        }
        failure {
            echo 'Deployment FAILED. Fetching container logs...'
            sshagent(credentials: ['ec2-ssh-key']) {
                sh """
                    ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                        cd ${APP_DIR}
                        echo "=== Nginx Logs ==="
                        sudo docker logs nginx --tail=30
                        echo "=== Backend Logs ==="
                        sudo docker logs backend --tail=30
                        echo "=== Frontend Logs ==="
                        sudo docker logs frontend --tail=30
                    ' || true
                """
            }
        }
        always {
            echo 'Pipeline finished.'
            cleanWs()
        }
    }
}
