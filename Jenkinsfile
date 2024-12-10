pipeline {
    agent any
    parameters {
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'Region of the S3 bucket')
        string(name: 'EC2_REGION', defaultValue: 'eu-west-1', description: 'Region of the EC2 instance')
        string(name: 'S3_BUCKET', defaultValue: 'statefile-remote-be', description: 'Name of the S3 bucket')
        string(name: 'EC2_HOST', defaultValue: 'ec2-63-33-156-75.eu-west-1.compute.amazonaws.com', description: 'EC2 instance hostname or IP')
        string(name: 'INSTANCE_ID', defaultValue: 'i-0d876e5f58afd60aa', description: 'EC2 instance ID')
        string(name: 'NETWORK', defaultValue: 'Mainnet', description: 'Name of Instance we are taking the snapshot from')
        string(name: 'VOLUME_SIZE', defaultValue: '1000', description: 'Size of EBS volume in GB')
    }
    environment {
        BACKUP_DIR = "/tmp/postgres_backup/snapshot"
        TAR_FILE = "/tmp/postgres_backup/postgres_backup.tar.gz"
        MOUNT_POINT = "/tmp/postgres_backup"
        DEVICE_NAME = "/dev/nvme2n1"
    }
    stages {
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
        always {
            sshagent(credentials: ['SSH_KEY_CRED']) {
                sh "ssh ubuntu@${EC2_HOST} 'sudo rm -rf ${BACKUP_DIR}/* ${TAR_FILE} && sudo umount ${MOUNT_POINT}'"
                sh """
                    aws ec2 detach-volume --volume-id ${env.VOLUME_ID} --region ${AWS_REGION}
                    aws ec2 delete-volume --volume-id ${env.VOLUME_ID} --region ${AWS_REGION}
                """
                echo "Detached and deleted EBS Volume: ${env.VOLUME_ID}"
            }
        }
    }
}