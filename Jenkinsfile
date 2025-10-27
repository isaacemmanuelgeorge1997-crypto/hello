pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'isaacgeorge/hello' 
        IMAGE_TAG      = "20251026-${BUILD_NUMBER}"

        EC2_HOST       = 'ec2-18-191-134-3.us-east-2.compute.amazonaws.com'
        EC2_USER       = 'isaac'
        EC2_CREDS      = 'ec2-creds'        // SSH private key credential ID
        APP_PORT       = '8080'
    }

    stages {

        stage('Checkout') {
            steps {
                echo "📥 Checking out source code..."
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image..."
                script {
                    bat """
                        docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo "📦 Pushing image to Docker Hub..."
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        bat """
                            echo Logging in to Docker Hub...
                            echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin
                            docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                            docker logout
                        """
                    }
                }
            }
        }

        stage('Deploy on EC2 (pull & run)') {
            steps {
                echo "🚀 Deploying to EC2: ${EC2_HOST}..."
                script {
                    withCredentials([sshUserPrivateKey(credentialsId: "${EC2_CREDS}", keyFileVariable: 'EC2_KEY')]) {
                        bat """
                            echo Setting correct permissions on private key...

                            where icacls
                            if exist "%EC2_KEY%" (
                                C:\\Windows\\System32\\icacls.exe "%EC2_KEY%" /inheritance:r
                                C:\\Windows\\System32\\icacls.exe "%EC2_KEY%" /grant:r "Administrators:F"
                                C:\\Windows\\System32\\icacls.exe "%EC2_KEY%" /grant:r "SYSTEM:F"
                                C:\\Windows\\System32\\icacls.exe "%EC2_KEY%" /remove "BUILTIN\\Users" "NT AUTHORITY\\Authenticated Users" "Everyone" >nul 2>nul
                            ) else (
                                echo ERROR: Key file not found: %EC2_KEY%
                                exit /b 1
                            )

                            echo Connecting and deploying on EC2...
                            ssh -T -o BatchMode=yes -o ConnectTimeout=25 -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "%EC2_KEY%" ${EC2_USER}@${EC2_HOST} "docker pull ${DOCKERHUB_REPO}:${IMAGE_TAG} && (docker rm -f hello || true) && docker run -d --name hello -p ${APP_PORT}:8080 ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful!"
        }
        failure {
            echo "❌ Deployment failed. Check Jenkins console output for details."
        }
    }
}
