
pipeline {
    agent any

    triggers {
        pollSCM('H/1 * * * *')
    }

    environment {
        AWS_REGION = 'ap-southeast-2'
        AWS_ACCOUNT_ID = '455012390845'
        ECR_REPOSITORY = 'my-docker-website'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t my-docker-website:%BUILD_NUMBER% .'
            }
        }

        stage('Test Website') {
            steps {
                bat 'docker rm -f website-test 2>NUL || exit /b 0'
                bat 'docker run -d --name website-test -p 8082:80 my-docker-website:%BUILD_NUMBER%'
                bat 'powershell -NoProfile -Command "Invoke-WebRequest http://localhost:8082 -UseBasicParsing | Out-Null"'
            }
        }

        stage('Push Image to AWS ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    bat '''
                    set AWS_DEFAULT_REGION=%AWS_REGION%
                    aws ecr get-login-password --region %AWS_REGION% | docker login --username AWS --password-stdin %AWS_ACCOUNT_ID%.dkr.ecr.%AWS_REGION%.amazonaws.com
                    docker tag my-docker-website:%BUILD_NUMBER% %AWS_ACCOUNT_ID%.dkr.ecr.%AWS_REGION%.amazonaws.com/%ECR_REPOSITORY%:%BUILD_NUMBER%
                    docker tag my-docker-website:%BUILD_NUMBER% %AWS_ACCOUNT_ID%.dkr.ecr.%AWS_REGION%.amazonaws.com/%ECR_REPOSITORY%:latest
                    docker push %AWS_ACCOUNT_ID%.dkr.ecr.%AWS_REGION%.amazonaws.com/%ECR_REPOSITORY%:%BUILD_NUMBER%
                    docker push %AWS_ACCOUNT_ID%.dkr.ecr.%AWS_REGION%.amazonaws.com/%ECR_REPOSITORY%:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            bat 'docker rm -f website-test 2>NUL || exit /b 0'
        }
        success {
            echo 'Website tested and image pushed to AWS ECR!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
