pipeline {
    agent any
    parameters {
        string(name: 'VOLUME_SIZE', defaultValue: '200', description: 'Size of EBS volume in GB')
    }
    environment {
        BACKUP_DIR = "/tmp/postgres_backup/snapshot"
        TAR_FILE = "/tmp/postgres_backup/postgres_backup.tar.gz"
        MOUNT_POINT = "/tmp/postgres_backup"
        VOLUME_NAME = "/dev/sda2"
        SLACK_CHANNEL = '#jenkins-notifications' 
        TOKEN_CREATION_TIME = ''
        TOKEN_TTL_SECONDS = 3600
    }
    stages {
         stage('Retrieve Secrets') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_REGION', variable: 'AWS_REGION'),
                    string(credentialsId: 'EC2_REGION', variable: 'EC2_REGION'),
                    string(credentialsId: 'S3_BUCKET', variable: 'S3_BUCKET'),
                    string(credentialsId: 'EC2_HOST', variable: 'EC2_HOST'),
                    string(credentialsId: 'INSTANCE_ID', variable: 'INSTANCE_ID'),
                    string(credentialsId: 'NETWORK', variable: 'NETWORK'),
                    string(credentialsId: 'SLACK', variable: 'SLACK'),
                    string(credentialsId: 'IAM_ROLE', variable: 'IAM_ROLE')
                ]) {
                    script {
                        // Set environment variables so they are available throughout the pipeline
                        env.AWS_REGION = "${AWS_REGION}"
                        env.EC2_REGION = "${EC2_REGION}"
                        env.S3_BUCKET = "${S3_BUCKET}"
                        env.EC2_HOST = "${EC2_HOST}"
                        env.INSTANCE_ID = "${INSTANCE_ID}"
                        env.NETWORK = "${NETWORK}"
                        env.SLACK = "${SLACK}"
                        env.IAM_ROLE = "${IAM_ROLE}"
                    }
                }
            }
        }
        stage('Retrieve AWS Session Token') {
            steps {
                script {
                    def getToken = {
                        def token = sh(script: '''
                            curl -X PUT -H "X-aws-ec2-metadata-token-ttl-seconds: ${TOKEN_TTL_SECONDS}" http://169.254.169.254/latest/api/token
                        ''', returnStdout: true).trim()

                        if (!token) {
                            error "Failed to retrieve session token."
                        }

                        echo "Session Token Retrieved"
                        env.AWS_METADATA_TOKEN = token
                        env.TOKEN_CREATION_TIME = System.currentTimeMillis().toString()
                        return token
                    }

                    // Initial token retrieval
                    getToken()
                }
            }
        }
        stage('Provision EBS Volume') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    script {
                        def creds = sh(script: """
                            curl --header "X-aws-ec2-metadata-token: ${env.AWS_METADATA_TOKEN}" http://169.254.169.254/latest/meta-data/iam/security-credentials/${IAM_ROLE}
                        """, returnStdout: true).trim()

                        // Extract AWS credentials
                        def accessKeyId = sh(script: "'${creds}' | jq -r .AccessKeyId", returnStdout: true).trim()
                        def secretAccessKey = sh(script: "'${creds}' | jq -r .SecretAccessKey", returnStdout: true).trim()
                        def sessionToken = sh(script: "'${creds}' | jq -r .Token", returnStdout: true).trim()

                        // Set the AWS credentials for the session
                        env.AWS_ACCESS_KEY_ID = accessKeyId
                        env.AWS_SECRET_ACCESS_KEY = secretAccessKey
                        env.AWS_SESSION_TOKEN = sessionToken

                        // Create EBS volume
                        withEnv([
                            "AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}",
                            "AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}",
                            "AWS_SESSION_TOKEN=${env.AWS_SESSION_TOKEN}",
                            "AWS_REGION=${AWS_REGION}"
                        ]) {
                            def volumeId = sh(script: """
                                aws ec2 create-volume --size ${params.VOLUME_SIZE} --volume-type gp2 --availability-zone us-east-1b --region ${AWS_REGION} --query 'VolumeId' --output text
                            """, returnStdout: true).trim()

                            echo "Created EBS Volume: ${volumeId}"

                            // Attach volume
                            sh """
                                sleep 10
                                aws ec2 attach-volume --volume-id ${volumeId} --instance-id ${INSTANCE_ID} --device ${VOLUME_NAME} --region ${AWS_REGION}
                            """
                            echo "Attached EBS Volume: ${volumeId} to instance: ${INSTANCE_ID}"

                            env.VOLUME_ID = volumeId
                        }
                    }
                }
            }
        }
        stage('Determine EBS Device Name') {
            steps {
                script {
                    // Fetch metadata token
                    def metadata_token = sh(script: '''
                        curl -X PUT -H "X-aws-ec2-metadata-token-ttl-seconds: 3600" http://169.254.169.254/latest/api/token
                    ''', returnStdout: true).trim()

                    // Fetch block device mappings
                    def block_devices = sh(script: """
                        curl --header "X-aws-ec2-metadata-token: ${metadata_token}" http://169.254.169.254/latest/meta-data/block-device-mapping/
                    """, returnStdout: true).trim()

                    // List all block devices with lsblk
                    def remote_device_info = sshagent(credentials: ['SSH_KEY_CRED']) {
                        retry(2) {
                            sh(script: """
                                ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} 'lsblk -o NAME,TYPE -J'
                            """, returnStdout: true).trim()
                        }
                    }

                    def nvme_devices = sh(script: """
                        echo '${remote_device_info}' | jq -r '.blockdevices[] | select(.name | test("^nvme")) | select(.name != "nvme0n1" and .name != "nvme1n1") | .name'
                    """, returnStdout: true).trim().split("\n")

                    // Assign the first device to DEVICE_NAME
                    def selectedDevice = "/dev/${nvme_devices[0]}"
                    echo "Selected device for mounting: ${selectedDevice}"

                    // Use withEnv to ensure the value is set properly
                    env.DEVICE_NAME = "/dev/${nvme_devices[0]}"

                    // Check again outside the withEnv block
                    if (!selectedDevice) {
                        error "Failed to assign DEVICE_NAME."
                    }

                    // Use DEVICE_NAME outside of withEnv to ensure it's passed to the next stage
                    echo "DEVICE_NAME in next scope: ${env.DEVICE_NAME}"
                }
            }
        }
        stage('Verify Device Name') {
            steps {
                script {
                    echo "DEVICE_NAME in Verify Stage: ${env.DEVICE_NAME}"
                    if (!env.DEVICE_NAME) {
                        error "DEVICE_NAME is not set correctly. Exiting..."
                    }
                }
            }
        }
        stage('Mount EBS Volume') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} \\
                        "sudo mkfs -t xfs ${env.DEVICE_NAME} &&
                        sudo mount ${env.DEVICE_NAME} ${MOUNT_POINT} &&
                        sudo chown -R ubuntu:ubuntu ${MOUNT_POINT} &&
                        sudo chmod -R 755 ${MOUNT_POINT} &&
                        echo 'EBS Successfully Mounted and permissions have been set.'"
                        """
                    }
                }
            }
        }
        stage('Prepare Snapshot Directory') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} \\
                        "mkdir -p ${BACKUP_DIR} && echo 'Backup directory created or already exists.'"
                        """
                    }
                }
            }
        }
        stage('Take PostgreSQL Snapshot') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(3) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} \\
                        "pg_basebackup -D ${BACKUP_DIR} -Ft -z -X stream --create-slot --slot=backup_slot -v"
                        """
                    }
                }
            }
        }
        stage('Verify Backup') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} \\
                        "test -d ${BACKUP_DIR} && echo 'Backup verification successful.'"
                        """
                    }
                }
            }
        }
        stage('Create Backup Tarball') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} \\
                        "tar -czf ${TAR_FILE} ${BACKUP_DIR} && echo 'Backup tarball created.'"
                        """
                    }
                }
            }
        }
        stage('Refresh Token'){
            steps {
                script {
                    // Fetch metadata token
                    def metadata_token = sh(script: '''
                        curl -X PUT -H "X-aws-ec2-metadata-token-ttl-seconds: 3600" http://169.254.169.254/latest/api/token
                    ''', returnStdout: true).trim()
                }
            }
        }
        stage('Upload Backup to S3') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(3) {
                        script {
                            def backupFile = sh(script: "echo postgres_backup_\$(date +%Y)", returnStdout: true).trim()
                            sh """
                            ssh ubuntu@${EC2_HOST} \\
                            "aws s3 cp ${TAR_FILE} s3://${S3_BUCKET}/${NETWORK}/${backupFile}.tar.gz --region ${AWS_REGION}"
                            """
                            env.BACKUP_FILE = backupFile
                        }
                    }
                }
            }
        }
        stage('Verify S3 Upload') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        script {
                            def s3_check = sh(script: """
                            ssh ubuntu@${EC2_HOST} \\
                            aws s3 ls s3://${S3_BUCKET}/${NETWORK}/${env.BACKUP_FILE}.tar.gz --region ${AWS_REGION}
                            """, returnStatus: true)
                            if (s3_check != 0) {
                                error "S3 upload verification failed."
                            }
                        }
                    }
                }
            }
        }
        stage('Remove All Replication Slots (Bash Loop)') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(5) {
                        sh """
                        sleep 5
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} \
                        "psql -d cexplorer -c \\\"SELECT pg_drop_replication_slot('backup_slot');\\\" && echo 'Replication slot removed successfully.'"
                        """
                    }
                }
            }
        }

    }
     post {
        success {
            script {
                def message = [
                    channel: "${SLACK_CHANNEL}",
                    text: "Build Success: ${env.JOB_NAME} [${env.BUILD_NUMBER}] (<${env.BUILD_URL}|Open>)",
                    color: 'good',
                    icon_emoji: ':tada:',  // Emoji for success
                    username: 'Jenkins'  // Custom bot name
                ]
                // Send message to Slack
                httpRequest(
                    url: "${SLACK}",
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: groovy.json.JsonOutput.toJson(message)
                )
            }
        }
        failure {
            script {
                def message = [
                    channel: "${SLACK_CHANNEL}",
                    text: "Build Failed: ${env.JOB_NAME} [${env.BUILD_NUMBER}] (<${env.BUILD_URL}|Open>)",
                    color: 'danger',
                    icon_emoji: ':warning:',  // Emoji for failure
                    username: 'Jenkins'  // Custom bot name
                ]
                // Send message to Slack
                httpRequest(
                    url: "${SLACK}",
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: groovy.json.JsonOutput.toJson(message)
                )
            }
        }
        always {
            sshagent(credentials: ['SSH_KEY_CRED']) {
                script {
                    // Clean up
                    sh """
                    ssh ubuntu@${EC2_HOST} \\
                    aws ec2 detach-volume --volume-id ${env.VOLUME_ID} --region ${AWS_REGION}
                    echo "Paused for 30 seconds..."
                    sleep 30 
                    aws ec2 delete-volume --volume-id ${env.VOLUME_ID} --region ${AWS_REGION} 
                    """
                    echo "Detached and deleted EBS Volume: ${env.VOLUME_ID}"
                }
            }
        }
    }
} 