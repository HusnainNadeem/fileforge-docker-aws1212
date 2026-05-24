pipeline {
    agent any

    environment {
        APP_DIR  = '/home/ubuntu/fileforge'
        EC2_USER = 'ubuntu'
        EC2_HOST = '18.222.164.24'
        COMPOSE  = 'docker-compose.yaml'
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/HusnainNadeem/fileforge-docker-aws1212.git'
            }
        }

        stage('Sync Files') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh "ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} 'mkdir -p ${APP_DIR}'"
                    sh """
                        rsync -avz -e "ssh -o StrictHostKeyChecking=no" \
                            --exclude='.git' \
                            --exclude='node_modules' \
                            --exclude='.env' \
                            ./ ${EC2_USER}@${EC2_HOST}:${APP_DIR}/
                    """
                }
            }
        }

        stage('Stop Old Containers') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            cd ${APP_DIR}
                            sudo docker compose -f ${COMPOSE} down --remove-orphans || true
                            sudo docker rm -f mongo backend frontend nginx 2>/dev/null || true
                            sudo docker ps -a
                        '
                    """
                }
            }
        }

        stage('Remove Old Images') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            sudo docker rmi backend frontend 2>/dev/null || true
                            sudo docker image prune -f
                            sudo docker images
                        '
                    """
                }
            }
        }

        stage('Build New Images') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            cd ${APP_DIR}
                            sudo docker compose -f ${COMPOSE} build --no-cache backend
                            sudo docker compose -f ${COMPOSE} build --no-cache frontend
                            sudo docker images
                        '
                    """
                }
            }
        }

        stage('Start New Containers') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            cd ${APP_DIR}
                            sudo docker compose -f ${COMPOSE} up -d
                            sudo docker compose -f ${COMPOSE} ps
                        '
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                sleep(time: 20, unit: 'SECONDS')
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            sudo docker compose -f ${APP_DIR}/${COMPOSE} ps
                            curl -sf http://localhost:80 > /dev/null && echo "SUCCESS: App is UP" || echo "FAILED: App not responding"
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment SUCCESS - http://${EC2_HOST}"
        }
        failure {
            echo 'Deployment FAILED - check logs below'
            sshagent(credentials: ['ec2-ssh-key']) {
                sh """
                    ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                        sudo docker logs nginx --tail=20 2>/dev/null || true
                        sudo docker logs backend --tail=20 2>/dev/null || true
                        sudo docker logs frontend --tail=20 2>/dev/null || true
                    '
                """
            }
        }
        always {
            cleanWs()
        }
    }
}
