pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'isaacgeorge/hello'
        IMAGE_TAG = "20251026-25"
        EC2_HOST = 'ec2-18-191-134-3.us-east-2.compute.amazonaws.com'
        EC2_USER = 'isaac'
        EC2_CREDS = 'ec2-creds' // Jenkins SSH private key credential ID
        APP_PORT = '9090'
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo '📦 Checking out code from GitHub...'
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image: ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                bat """
                    docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                """
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                echo "⬆️ Pushing image to Docker Hub..."
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    bat """
                        docker login -u %DOCKER_USER% -p %DOCKER_PASS%
                        docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                        docker logout
                    """
                }
            }
        }

        stage('Deploy on EC2 (pull & run)') {
            steps {
                echo "🚀 Deploying on EC2: ${EC2_HOST}..."
                script {
                    withCredentials([sshUserPrivateKey(credentialsId: "${EC2_CREDS}", keyFileVariable: 'EC2_KEY')]) {
                        bat """
                            echo Setting correct permissions on private key...
                            if exist "%EC2_KEY%" (
                                icacls "%EC2_KEY%" /inheritance:r
                                icacls "%EC2_KEY%" /grant:r "Administrators:F"
                                icacls "%EC2_KEY%" /remove "BUILTIN\\Users" "NT AUTHORITY\\Authenticated Users" "Everyone" >nul 2>nul
                            ) else (
                                echo ERROR: Key file not found: %EC2_KEY%
                            )

                            echo Deploying to EC2: ${EC2_HOST}...
                            ssh -T -o BatchMode=yes -o ConnectTimeout=20 -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "%EC2_KEY%" ${EC2_USER}@${EC2_HOST} "docker pull ${DOCKERHUB_REPO}:${IMAGE_TAG} && (docker rm -f hello || true) && docker run -d --name hello -p ${APP_PORT}:8080 ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployed successfully!"
            echo "🌐 Visit: http://${EC2_HOST}:${APP_PORT}"
        }
        failure {
            echo "❌ Deployment failed. Check Jenkins console output for details."
        }
    }
}
