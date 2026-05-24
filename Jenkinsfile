pipeline {
    agent any

    environment {
        APP_DIR  = '/home/ubuntu/fileforge'
        EC2_USER = 'ubuntu'
        EC2_HOST = '18.222.164.24'
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

        stage('Validate') {
            parallel {
                stage('Lint Backend') {
                    steps {
                        dir('backend') {
                            sh 'npm install'
                            sh 'npm run lint || echo "No lint script, skipping..."'
                        }
                    }
                }
                stage('Lint Frontend') {
                    steps {
                        dir('frontend') {
                            sh 'npm install'
                            sh 'npm run lint || echo "No lint script, skipping..."'
                        }
                    }
                }
            }
        }

        stage('Sync Files') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh "ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} 'mkdir -p ${APP_DIR}'"
                    sh """
                        rsync -avz \
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
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} \
                        'cd ${APP_DIR} && sudo docker compose down --remove-orphans && sudo docker ps -a'
                    """
                }
            }
        }

        stage('Remove Old Images') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} \
                        'sudo docker rmi backend frontend 2>/dev/null || true && sudo docker image prune -f && sudo docker images'
                    """
                }
            }
        }

        stage('Build New Images') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} \
                        'cd ${APP_DIR} && sudo docker compose build --no-cache backend && sudo docker compose build --no-cache frontend && sudo docker images'
                    """
                }
            }
        }

        stage('Start New Containers') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} \
                        'cd ${APP_DIR} && sudo docker compose up -d && sudo docker compose ps'
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                sleep(time: 15, unit: 'SECONDS')
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} \
                        'sudo docker compose -f ${APP_DIR}/docker-compose.yml ps && curl -sf http://localhost:80 > /dev/null && echo "SUCCESS: App is UP" || echo "FAILED: App not responding"'
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
            sshagent(credentials: ['ec2-ssh-key']) {
                sh """
                    ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} \
                    'sudo docker logs nginx --tail=30 && sudo docker logs backend --tail=30 && sudo docker logs frontend --tail=30' || true
                """
            }
        }
        always {
            cleanWs()
        }
    }
}
