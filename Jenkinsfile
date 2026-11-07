
pipeline {
    agent any


    environment {
        AWS_DEFAULT_REGION = 'ap-southeast-2'
        ECR_REPOSITORY = '455012390845.dkr.ecr.ap-southeast-2.amazonaws.com/my-docker-website'
        AWS_CLI = '"C:\\Program Files\\Amazon\\AWSCLIV2\\aws.exe"'
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
                    bat 'docker rm -f website-test 2>NUL || exit /b 0'

                    bat '%AWS_CLI% ecr get-login-password --region %AWS_DEFAULT_REGION% | docker login --username AWS --password-stdin 455012390845.dkr.ecr.ap-southeast-2.amazonaws.com'

                    bat 'docker tag my-docker-website:%BUILD_NUMBER% %ECR_REPOSITORY%:%BUILD_NUMBER%'
                    bat 'docker tag my-docker-website:%BUILD_NUMBER% %ECR_REPOSITORY%:latest'

                    bat 'docker push %ECR_REPOSITORY%:%BUILD_NUMBER%'
                    bat 'docker push %ECR_REPOSITORY%:latest'
                }
            }
        }
    }

    post {
        always {
            bat 'docker rm -f website-test 2>NUL || exit /b 0'
        }

        success {
            echo 'Docker image successfully pushed to AWS ECR!'
        }

        failure {
            echo 'Pipeline failed. Check Console Output.'
        }
    }
}
