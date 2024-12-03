pipeline {
    agent any
    parameters {
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'Region of the S3 bucket')
        string(name: 'EC2_REGION', defaultValue: 'us-east-1', description: 'Region of the EC2 instance')
        string(name: 'S3_BUCKET', defaultValue: 'statefile-remote-be', description: 'Name of the S3 bucket')
        string(name: 'EC2_HOST', defaultValue: 'ec2-100-27-119-149.compute-1.amazonaws.com', description: 'EC2 instance hostname or IP')
        string(name: 'NETWORK', defaultValue: 'Sanchonet', description: 'Name of Instance we are taking the snapshot from')
    }
    environment {
        BACKUP_DIR = "/tmp/postgres_backup"
        TAR_FILE = "/tmp/postgres_backup.tar.gz"
    }
    stages {
        stage('Checkout Code from Git') {
            steps {
                git branch: 'main', url: 'git@github.com:aaboyle878/PostGresSnapShot.git'
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
        stage('Prepare Backup Directory') {
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
                        "pg_basebackup -D ${BACKUP_DIR} -Ft -z -X fetch -v"
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
                retry(2) {
                    script {
                        // Install aws-cli if not present
                        sh '''
                        if ! command -v aws &> /dev/null
                        then
                            echo "aws-cli not found, installing..."
                            sudo apt-get update
                            sudo apt-get install -y awscli
                        fi
                        '''
                        def s3_check = sh(script: """
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
    post {
        always {
            sshagent(credentials: ['SSH_KEY_CRED']) {
                sh "ssh ubuntu@${EC2_HOST} 'rm -rf ${BACKUP_DIR} ${TAR_FILE}'"
            }
        }
    }
}
