pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'isaacgeorge/hello'
        IMAGE_TAG      = "20251026-${env.BUILD_NUMBER}"

        EC2_HOST = 'ec2-18-191-134-3.us-east-2.compute.amazonaws.com'
        EC2_USER = 'isaac'
        EC2_CREDS = 'ec2-creds'   // Jenkins SSH private key credential ID
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out source code..."
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    bat """
                        docker build -t %DOCKERHUB_REPO%:%IMAGE_TAG% .
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    echo "Pushing Docker image to DockerHub..."
                    bat """
                        docker login -u isaacgeorge -p %DOCKERHUB_TOKEN%
                        docker push %DOCKERHUB_REPO%:%IMAGE_TAG%
                    """
                }
            }
        }

        stage('Deploy on EC2 (pull & run)') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: "${env.EC2_CREDS}", keyFileVariable: 'EC2_KEY')]) {
                    script {
                        echo "Setting correct permissions on private key..."
                        bat """
                            if exist "%EC2_KEY%" (
                                icacls "%EC2_KEY%" /inheritance:r
                                icacls "%EC2_KEY%" /grant:r "Administrators:F"
                                icacls "%EC2_KEY%" /remove "BUILTIN\\Users" "NT AUTHORITY\\Authenticated Users" "Everyone" >nul 2>nul
                            ) else (
                                echo ERROR: Key file not found: %EC2_KEY%
                                exit /b 1
                            )
                        """

                        echo "Deploying to EC2: ${env.EC2_HOST}..."
                        bat """
                            ssh -T -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "%EC2_KEY%" %EC2_USER%@%EC2_HOST% ^
                            "docker pull %DOCKERHUB_REPO%:%IMAGE_TAG% && (docker rm -f hello || true) && docker run -d --name hello -p 9090:8080 %DOCKERHUB_REPO%:%IMAGE_TAG%"
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployed successfully!"
            echo "Visit: http://${env.EC2_HOST}:9090"
        }
        failure {
            echo "❌ Deployment failed. Check Jenkins console output for details."
        }
    }
}
