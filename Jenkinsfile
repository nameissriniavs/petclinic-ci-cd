pipeline {
  agent any

  stages {

    stage('Build') {
      steps {
        sh '''
          set -e
          cd initial
          ./mvnw package

          # Create fixed artifact name expected by S3 + systemd service
          cp target/*.jar target/petclinicapp-v1.jar
          ls -l target/
        '''
      }
    }

    stage('Upload to S3') {
      steps {
        sh '''
          set -e
          S3_BUCKET=petclinicapp2026
          BUILD_FILE_NAME=petclinicapp-v1.jar

          aws s3 cp $WORKSPACE/initial/target/$BUILD_FILE_NAME s3://$S3_BUCKET/$BUILD_FILE_NAME
        '''
      }
    }

    stage('Deploy to EC2') {
      steps {
        sh '''
          set -e
          USER=petclinicapp
          S3_BUCKET=petclinicapp2026
          BUILD_FILE_NAME=petclinicapp-v1.jar
          LOCAL_FILE_PATH=/home/$USER/$BUILD_FILE_NAME

          output=$(aws ec2 describe-instances \
            --filters "Name=tag:appname,Values=petclinic" "Name=instance-state-name,Values=running" \
            --query "Reservations[*].Instances[*].[InstanceId,PublicIpAddress]" \
            --output json)

          ips=$(echo "$output" | jq -r '.[][][1]')
          echo "$ips"

          for ip in $ips; do
            echo "Connecting to $ip"
            ssh -o StrictHostKeyChecking=no -i /home/jenkins/petclinicappkey.pem ubuntu@$ip << EOF
              sudo aws s3 cp s3://$S3_BUCKET/$BUILD_FILE_NAME $LOCAL_FILE_PATH
              sudo systemctl restart petclinicapp.service
              sudo systemctl status petclinicapp.service --no-pager
EOF
          done
        '''
      }
    }
  }
}

