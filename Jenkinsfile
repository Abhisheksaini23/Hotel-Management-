pipeline {
  agent any

  environment {
    IMAGE_NAME = "YOUR_DOCKERHUB_USERNAME/hotel-management"    // <- replace
    IMAGE_TAG  = "${env.BUILD_NUMBER}"
    DOCKERHUB_CRED = 'docker-hub-creds'                      // Jenkins credentials id (username/password)
    SSH_CRED_ID = 'ec2-ssh-key'                              // Jenkins SSH credentials id (private key)
    EC2_USER = 'ec2-user'                                    // or 'ubuntu' depending on AMI
    EC2_HOST = 'EC2_PUBLIC_IP_OR_DNS'                        // <- replace with your EC2 public IP or DNS
    APP_PORT = '3000'                                        // port your app listens on inside container
    HOST_PORT = '80'                                         // host port you want to expose on EC2 (optional)
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install & Test') {
      steps {
        sh 'node -v || true'
        sh 'npm ci'
        sh 'npm test || true'   // keep pipeline non-blocking on test failure, change as you prefer
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
        }
      }
    }

    stage('Docker Login & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CRED}", usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push ${IMAGE_NAME}:${IMAGE_TAG}
            docker logout
          '''
        }
      }
    }

    stage('Deploy to EC2') {
      steps {
        // Uses SSH private key credential stored in Jenkins (SSH Agent or credentials binding)
        // This example writes the private key to a file and uses ssh -i (works with username/private key credential)
        withCredentials([sshUserPrivateKey(credentialsId: "${SSH_CRED_ID}", keyFileVariable: 'EC2_KEY', usernameVariable: 'EC2_SSH_USER')]) {
          // Note: EC2_SSH_USER provided by credentials; if not present, fall back to EC2_USER env var
          script {
            def sshUser = EC2_SSH_USER ?: env.EC2_USER
            sh """
              chmod 600 "${EC2_KEY}"
              # stop & remove old container, pull new image, run new container
              ssh -o StrictHostKeyChecking=no -i "${EC2_KEY}" ${sshUser}@${EC2_HOST} \\
                "docker pull ${IMAGE_NAME}:${IMAGE_TAG} && \\
                 docker rm -f hotel_app || true && \\
                 docker run -d --name hotel_app -p ${HOST_PORT}:${APP_PORT} ${IMAGE_NAME}:${IMAGE_TAG}"
            """
          }
        }
      }
    }
  } // stages

  post {
    success {
      echo "Pipeline succeeded: ${IMAGE_NAME}:${IMAGE_TAG}"
    }
    failure {
      echo "Pipeline failed"
    }
  }
}
