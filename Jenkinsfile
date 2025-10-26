pipeline {
  agent any

  environment {
    DOCKERHUB_REPO = 'isaacgeorge/hello' 
    IMAGE_TAG      =  new Date().format('yyyyMMdd-HHmm')
    DOCKERHUB_CRED = 'dockerhub-creds'

    EC2_HOST       = 'ec2-18-191-134-3.us-east-2.compute.amazonaws.com'
    EC2_USER       = 'isaac'
    EC2_CREDS      = 'ec2-creds'                
    APP_PORT       = '8080'
  }
  
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Docker image') {
      steps {
        script {
          // Ensure Docker is available on the agent (Docker Desktop on Windows or Docker Engine on Linux)
          if (isUnix()) {
            sh "docker version"
            sh "docker build -t ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG} ."
          } else {
            bat "docker version"
            bat "docker build -t ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG} ."
          }
        }
      }
    }

    stage('Push Docker image') {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CRED, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
            if (isUnix()) {
              sh """
                echo "\$DH_PASS" | docker login -u "\$DH_USER" --password-stdin
                docker push ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG}
              """
            } else {
              bat """
                docker login -u %DH_USER% -p %DH_PASS%
                docker push ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG}
              """
            }
          }
        }
      }
    }
    
    stage('Deploy on EC2 (pull & run)') {
        when { expression { return env.EC2_HOST?.trim() } }
        steps {
            // Jenkins credential must be "SSH Username with private key"
            withCredentials([sshUserPrivateKey(credentialsId: env.EC2_CREDS,
                                      keyFileVariable: 'EC2_KEY',
                                      usernameVariable: 'EC2_USER_FROM_CREDS')]) {
            script {
               def user = (env.EC2_USER?.trim()) ?: EC2_USER_FROM_CREDS
               def remote = "${user}@${env.EC2_HOST}"

               if (isUnix()) {
                   sh "chmod 600 \"$EC2_KEY\" && ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i \"$EC2_KEY\" ${remote} 'echo connected'"
                   sh """
                      ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "$EC2_KEY" ${remote} \\
                      'docker pull ${DOCKERHUB_REPO}:${IMAGE_TAG} && \\
                       docker rm -f hello || true && \\
                       docker run -d --name hello -p 80:${APP_PORT} ${DOCKERHUB_REPO}:${IMAGE_TAG}'
                      """
                } else {
                   bat """
                       set "KEY=%EC2_KEY%"
                       for /f "tokens=* delims=" %%I in ('whoami') do set "WHO=%%I"
                       icacls "%KEY%" /inheritance:r
                       icacls "%KEY%" /grant:r "%WHO%:F"
                       icacls "%KEY%" /remove "BUILTIN\\Users" "NT AUTHORITY\\Authenticated Users" "Everyone" 1>nul 2>nul
        
                       ssh -T -o BatchMode=yes -o ConnectTimeout=15 -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "%EC2_KEY%" ${remote} "docker pull ${DOCKERHUB_REPO}:${IMAGE_TAG} && (docker rm -f hello || true) && docker run -d --name hello -p 9090:${APP_PORT} ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                       """
                }
            }
        }
      }
    }
  }
  post {
    success { echo "Deployed: http://${EC2_HOST}:9090/" }
    always  { echo 'Pipeline finished.' }
  }
}
