pipeline {
  agent any

  environment {
    // ----- CONFIGURATION -----
    DOCKERHUB_REPO = 'isaacgeorge/hello'                   // your Docker Hub repo
    IMAGE_TAG      = "20251026-${BUILD_NUMBER}"            // unique tag per build
    EC2_HOST       = 'ec2-18-191-134-3.us-east-2.compute.amazonaws.com'
    EC2_USER       = 'isaac'
    EC2_CREDS      = 'ec2-creds'                           // SSH private key credential ID in Jenkins
    APP_PORT       = '9090'                                // public port on EC2
  }

  stages {

    stage('Checkout') {
      steps {
        echo "Checking out code from Git..."
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          echo "Building Docker image: ${DOCKERHUB_REPO}:${IMAGE_TAG}"
          bat """
            docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
          """
        }
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          script {
            echo "Pushing image to Docker Hub..."
            bat """
              docker login -u %DOCKER_USER% -p %DOCKER_PASS%
              docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
              docker logout
            """
          }
        }
      }
    }

    stage('Deploy on EC2 (pull & run)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: "${EC2_CREDS}", keyFileVariable: 'KEY')]) {
          script {
            bat """
              echo Setting correct permissions on private key...
              if exist "%KEY%" (
                "C:\\Windows\\System32\\icacls.exe" "%KEY%" /inheritance:r
                "C:\\Windows\\System32\\icacls.exe" "%KEY%" /grant:r "%USERNAME%:F"
                "C:\\Windows\\System32\\icacls.exe" "%KEY%" /remove "BUILTIN\\Users" "NT AUTHORITY\\Authenticated Users" "Everyone" 1>nul 2>nul
              ) else (
                echo ERROR: Key file not found: %KEY%
              )

              echo Deploying to EC2: ${EC2_HOST}...
              ssh -T -o BatchMode=yes -o ConnectTimeout=15 -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "%KEY%" ${EC2_USER}@${EC2_HOST} "docker pull ${DOCKERHUB_REPO}:${IMAGE_TAG} && (docker rm -f hello || true) && docker run -d --name hello -p ${APP_PORT}:8080 ${DOCKERHUB_REPO}:${IMAGE_TAG}"
            """
          }
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline finished."
    }
    success {
      echo "Deployed: http://${EC2_HOST}:${APP_PORT}"
    }
    failure {
      echo "Deployment failed. Check permissions or credentials."
    }
  }
}
