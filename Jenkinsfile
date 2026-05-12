pipeline {
  agent any

  environment {
    AWS_REGION      = "us-east-1"
    S3_BUCKET       = "petclinicapp-sriniavsaps007"
    BUILD_FILE_NAME = "petclinicapp-v1.jar"

    APP_TAG_KEY     = "appname"
    APP_TAG_VALUE   = "petclinic"

    // ✅ NEW: Jenkins server generated SSH key
    SSH_KEY         = "/home/jenkins/.ssh/petclinickey"
    SSH_USER        = "ubuntu"

    REMOTE_USER     = "petclinicapp"
    REMOTE_PATH     = "/home/petclinicapp/petclinicapp-v1.jar"
    SERVICE_NAME    = "petclinicapp.service"
  }

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build (Maven)') {
      steps {
        sh '''
          set -euo pipefail
          cd initial
          chmod +x mvnw
          ./mvnw -q clean package -DskipTests

          echo "Build output:"
          ls -lh target/${BUILD_FILE_NAME}
        '''
      }
    }

    stage('Upload Artifact to S3') {
      steps {
        sh '''
          set -euo pipefail
          aws --region ${AWS_REGION} s3 cp "$WORKSPACE/initial/target/${BUILD_FILE_NAME}" "s3://${S3_BUCKET}/${BUILD_FILE_NAME}"
          echo "S3 content:"
          aws --region ${AWS_REGION} s3 ls "s3://${S3_BUCKET}/"
        '''
      }
    }

    stage('Deploy to EC2 Instances (tag appname=petclinic)') {
      steps {
        sh '''
          set -euo pipefail

          IPS=$(aws --region ${AWS_REGION} ec2 describe-instances \
            --filters "Name=tag:${APP_TAG_KEY},Values=${APP_TAG_VALUE}" "Name=instance-state-name,Values=running" \
            --query "Reservations[*].Instances[*].PublicIpAddress" \
            --output text)

          echo "Target EC2 IPs: ${IPS}"

          if [ -z "${IPS}" ]; then
            echo "ERROR: No running EC2 instances found with tag ${APP_TAG_KEY}=${APP_TAG_VALUE}"
            exit 1
          fi

          for ip in ${IPS}; do
            echo "-----------------------------------------------------"
            echo "Deploying to ${ip}"

            ssh -o StrictHostKeyChecking=no -i "${SSH_KEY}" "${SSH_USER}@${ip}" << EOF
              set -euo pipefail
              sudo aws --region ${AWS_REGION} s3 cp "s3://${S3_BUCKET}/${BUILD_FILE_NAME}" "${REMOTE_PATH}"
              sudo chown ${REMOTE_USER}:${REMOTE_USER} "${REMOTE_PATH}" || true
              sudo systemctl restart ${SERVICE_NAME}
              sudo systemctl status ${SERVICE_NAME} --no-pager -l | head -n 30
EOF
          done
        '''
      }
    }
  }

  post {
    success { echo "✅ Pipeline completed successfully." }
    failure { echo "❌ Pipeline failed. Check console output of failed stage." }
  }
}
