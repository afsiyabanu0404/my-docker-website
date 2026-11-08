
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

        stage('Deploy Local Website') {
            steps {
                bat 'docker rm -f my-website 2>NUL || exit /b 0'
                bat 'docker run -d --name my-website --restart unless-stopped -p 8081:80 my-docker-website:%BUILD_NUMBER%'
                bat 'powershell -NoProfile -Command "Invoke-WebRequest http://localhost:8081 -UseBasicParsing | Out-Null"'
            }
        }

        stage('Deploy to AWS EC2') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    powershell '''
                        $ErrorActionPreference = 'Stop'

                        $aws = 'C:\\Program Files\\Amazon\\AWSCLIV2\\aws.exe'
                        $instance = 'i-07777f3f9b78ef0fb'
                        $repository = '455012390845.dkr.ecr.ap-southeast-2.amazonaws.com/my-docker-website'

                        $commands = @(
                            "aws ecr get-login-password --region ap-southeast-2 | sudo docker login --username AWS --password-stdin 455012390845.dkr.ecr.ap-southeast-2.amazonaws.com",
                            "sudo docker pull ${repository}:latest",
                            "sudo docker rm -f my-website || true",
                            "sudo docker run -d --name my-website --restart unless-stopped -p 80:80 ${repository}:latest",
                            "sleep 5",
                            "curl -f http://localhost"
                        )

                        $payload = @{
                            DocumentName = 'AWS-RunShellScript'
                            InstanceIds = @($instance)
                            Parameters = @{
                                commands = $commands
                            }
                            Comment = "Website deployment from Jenkins build $env:BUILD_NUMBER"
                        } | ConvertTo-Json -Depth 6

                        $jsonPath = Join-Path $env:WORKSPACE 'ssm-command.json'
                        Set-Content -Path $jsonPath -Value $payload -Encoding ascii

                       $fileUri = "file://$jsonPath"

                        Write-Host 'Sending deployment command to EC2...'

                        $response = & $aws ssm send-command --region ap-southeast-2 --cli-input-json $fileUri | ConvertFrom-Json

                        if ($LASTEXITCODE -ne 0) {
                            throw 'Could not send deployment command to EC2.'
                        }

                        $commandId = $response.Command.CommandId
                        Write-Host "SSM Command ID: $commandId"

                        $completed = $false

                        for ($i = 0; $i -lt 30; $i++) {
                            Start-Sleep -Seconds 2

                            $status = & $aws ssm get-command-invocation `
                                --region ap-southeast-2 `
                                --command-id $commandId `
                                --instance-id $instance `
                                --query Status `
                                --output text 2>$null

                            if ($LASTEXITCODE -eq 0) {
                                Write-Host "EC2 deployment status: $status"

                                if ($status -eq 'Success') {
                                    $completed = $true
                                    break
                                }

                                if ($status -in @('Failed', 'TimedOut', 'Cancelled', 'Cancelling')) {
                                    $details = & $aws ssm get-command-invocation `
                                        --region ap-southeast-2 `
                                        --command-id $commandId `
                                        --instance-id $instance `
                                        --query StandardErrorContent `
                                        --output text

                                    Write-Host $details
                                    throw "EC2 deployment failed: $status"
                                }
                            }
                        }

                        if (-not $completed) {
                            throw 'EC2 deployment did not finish within 60 seconds. Check SSM command status.'
                        }

                        Write-Host 'EC2 WEBSITE DEPLOYED SUCCESSFULLY!'
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
            echo 'Build, test, ECR push, local deployment and AWS EC2 deployment completed!'
        }
        failure {
            echo 'Pipeline failed. Check Console Output.'
        }
    }
}
