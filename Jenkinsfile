pipeline {
    agent any

    stages {
        stage('build') {
            steps {
                sh '''
                cd initial
                ./mvnw package
                
                '''
            }
        }
        stage('Copy') {
            steps {
                sh '''
                USER=petclinicapp
                GROUP=petclinicapp
                S3_BUCKET=petclinicapp
                BUILD_FILE_NAME=petclinicapp-v1.jar
                LOCAL_FILE_PATH=/home/$USER/$BUILD_FILE_NAME
                aws s3 cp $WORKSPACE/initial/target/$BUILD_FILE_NAME s3://$S3_BUCKET
                       
                
                output=$(aws ec2 describe-instances \
                --filter "Name=tag:appname,Values=petclinic" "Name=instance-state-name,Values=running" \
                --query "Reservations[*].Instances[*].[InstanceId,PublicIpAddress]" \
                --output json)

                ips=$(echo "$output" | jq -r '.[][][1]')
                echo "$ips"

                 for ip in $ips; do
                    echo "-----------------------------------------------------"
                    echo "Connecting to $ip"
                    ssh  -o StrictHostKeyChecking=no -i /home/jenkins/petclinicappkey.pem ubuntu@$ip << EOF
                    sudo aws s3 cp s3://$S3_BUCKET/$BUILD_FILE_NAME $LOCAL_FILE_PATH
                    sudo systemctl restart petclinicapp.service
                    echo "Running command on $ip"
                    echo "-----------------------------------------------------"
EOF
                done
                '''
            }
        }
    }
}
