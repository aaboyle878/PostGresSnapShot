pipeline {
    agent any
    parameters {
        string(name: 'VOLUME_SIZE', defaultValue: '0', description: 'Size of EBS volume in GB')
    }
    environment {
        BACKUP_DIR = "/tmp/postgres_backup/snapshot"
        TAR_FILE = "/tmp/postgres_backup/postgres_backup.tar.gz"
        MOUNT_POINT = "/tmp/postgres_backup"
        DEVICE_NAME = "/dev/nvme2n1"
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
                    string(credentialsId: 'NETWORK', variable: 'NETWORK')
                ]) {
                    script {
                        // Print masked values for debugging (optional)
                        echo "AWS Region: ${AWS_REGION} (retrieved from secret)"
                        echo "AWS Region: ${EC2_REGION} (retrieved from secret)"
                        echo "S3 Bucket: ${S3_BUCKET} (retrieved from secret)"
                        echo "EC2 Host: ${EC2_HOST} (retrieved from secret)"
                        echo "Instance ID: ${INSTANCE_ID} (retrieved from secret)"
                        echo "AWS Region: ${NETWORK} (retrieved from secret)"
                    }
                }
            }
        }
        stage('SSH to EC2 Instance') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} "echo 'Connected to EC2 instance.'"
                        """
                    }
                }
            }
        }
        stage('Prepare Mount Directory') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} \\
                        "mkdir -p ${MOUNT_POINT} && echo 'Backup directory created or already exists.'"
                        """
                    }
                }
            }
        }
        stage('Provision EBS Volume') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    script {
                        def volumeId = sh(script: """
                            aws ec2 create-volume --size ${params.VOLUME_SIZE} --volume-type gp2 --availability-zone eu-west-1b --region ${AWS_REGION} --query 'VolumeId' --output text
                        """, returnStdout: true).trim()
                        echo "Created EBS Volume: ${volumeId}"
                        sh """
                            aws ec2 attach-volume --volume-id ${volumeId} --instance-id ${params.INSTANCE_ID} --device ${DEVICE_NAME} --region ${AWS_REGION}
                        """
                        echo "Attached EBS Volume: ${volumeId} to instance: ${params.INSTANCE_ID}"
                        env.VOLUME_ID = volumeId
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
                        "sudo mkfs -t xfs ${DEVICE_NAME} &&
                        sudo mount ${DEVICE_NAME} ${MOUNT_POINT} &&
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
        stage('Remove Replication Slot') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} \\
                        "psql -d cexplorer -c 'SELECT pg_drop_replication_slot(\'backup_slot\');' && echo 'Replication slot removed successfully.'"
                        """
                    }
                }
            }
        }
    }
    post {
        success {
            slackSend(
                channel: '#jenkins-notifications',
                color: 'good',
                message: "Build Success: ${env.JOB_NAME} [${env.BUILD_NUMBER}] (<${env.BUILD_URL}|Open>)"
            )
        }
        failure {
            slackSend(
                channel: '#jenkins-notifications',
                color: 'danger',
                message: "Build Failed: ${env.JOB_NAME} [${env.BUILD_NUMBER}] (<${env.BUILD_URL}|Open>)"
            )
        }
        always {
            sshagent(credentials: ['SSH_KEY_CRED']) {
                sh "ssh ubuntu@${EC2_HOST} 'sudo rm -rf ${BACKUP_DIR}/* ${TAR_FILE} && sudo umount ${MOUNT_POINT}'"
                sh """
                    aws ec2 detach-volume --volume-id ${env.VOLUME_ID} --region ${AWS_REGION}
                    aws ec2 delete-volume --volume-id ${env.VOLUME_ID} --region ${AWS_REGION}
                """
                echo "Detached and deleted EBS Volume: ${env.VOLUME_ID}"
            }
            slackSend(
                channel: '#jenkins-notifications',
                color: 'warning',
                message: "Build Completed: ${env.JOB_NAME} [${env.BUILD_NUMBER}] (<${env.BUILD_URL}|Open>)"
            )
        }
    }
}