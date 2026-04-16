pipeline {
  agent any

  environment {
    JAVA_HOME = "/usr/lib/jvm/java-21-openjdk-amd64"
    PATH = "${env.JAVA_HOME}/bin:${env.PATH}"

    S3_BUCKET = "petclinicapp2026-003713966195-us-east-1-an"
    BUILD_FILE_NAME = "petclinicapp-v1.jar"

    APP_USER = "petclinicapp"
    LOCAL_FILE_PATH = "/home/petclinicapp/petclinicapp-v1.jar"

    SSH_KEY = "/home/jenkins/petclinicappkey"

    TAG_KEY = "appname"
    TAG_VALUE = "petclinic"
  }

  stages {

    stage('Build') {
      steps {
        sh '''
          set -e
          cd initial

          chmod +x ./mvnw
          ./mvnw clean package

          # Remove old fixed-name jar (prevents cp error when multiple jars exist)
          rm -f target/${BUILD_FILE_NAME}

          # Pick the newest real jar; exclude .original and the fixed-name jar
          JAR=$(ls -1t target/*.jar | grep -vE '(.original|'"${BUILD_FILE_NAME}"')$' | head -n 1)

          echo "Built jar detected: $JAR"
          cp "$JAR" target/${BUILD_FILE_NAME}

          echo "Target directory contents:"
          ls -l target/
        '''
      }
    }

    stage('Upload to S3') {
      steps {
        sh '''
          set -e
          aws s3 cp $WORKSPACE/initial/target/${BUILD_FILE_NAME} s3://${S3_BUCKET}/${BUILD_FILE_NAME}
          echo "Uploaded to: s3://${S3_BUCKET}/${BUILD_FILE_NAME}"
        '''
      }
    }

    stage('Deploy to Application EC2') {
      steps {
        sh '''
          set -e

          output=$(aws ec2 describe-instances \
            --filters "Name=tag:${TAG_KEY},Values=${TAG_VALUE}" "Name=instance-state-name,Values=running" \
            --query "Reservations[*].Instances[*].[InstanceId,PublicIpAddress]" \
            --output json)

          ips=$(echo "$output" | jq -r '.[][][1]')
          echo "Target IPs: $ips"

          for ip in $ips; do
            echo "-----------------------------------------------------"
            echo "Connecting to $ip"

            ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ubuntu@$ip << EOF
              sudo aws s3 cp s3://${S3_BUCKET}/${BUILD_FILE_NAME} ${LOCAL_FILE_PATH}
              sudo chown ${APP_USER}:${APP_USER} ${LOCAL_FILE_PATH}
              sudo chmod 644 ${LOCAL_FILE_PATH}
              sudo systemctl restart petclinicapp.service
              sudo systemctl status petclinicapp.service --no-pager
EOF

            echo "-----------------------------------------------------"
          done
        '''
      }
    }
  }
}
