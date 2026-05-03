pipeline {
    agent any

    environment {
        IMAGE_NAME = "fileforge-app"
        CONTAINER_NAME = "fileforge-container"
        PORT = "80"
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "Cloning repository..."
                git branch: 'main', url: 'https://github.com/HusnainNadeem/fileforge-docker-aws.git'
            }
        }

        stage('Stop Old Container') {
            steps {
                script {
                    sh '''
                    docker stop $CONTAINER_NAME || true
                    docker rm $CONTAINER_NAME || true
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t $IMAGE_NAME ."
                }
            }
        }

        stage('Run Container') {
            steps {
                script {
                    sh """
                    docker run -d -p $PORT:80 --name $CONTAINER_NAME $IMAGE_NAME
                    """
                }
            }
        }

    }

    post {
        success {
            echo "🚀 Deployment Successful!"
        }
        failure {
            echo "❌ Deployment Failed!"
        }
    }
}
