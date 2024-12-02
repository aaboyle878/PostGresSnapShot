pipeline {
    agent any
    environment {
        AWS_REGION = 'your-aws-region' // e.g., us-west-2
        S3_BUCKET = 'your-s3-bucket-name'
        EC2_HOST = 'your-ec2-instance-ip'
        SSH_KEY_CRED = 'jenkins-ssh-key-id' // Jenkins credentials for SSH
    }
    stages {
        stage('SSH to EC2 Instance') {
            steps {
                sshagent(credentials: [env.SSH_KEY_CRED]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} "echo 'Connected to EC2 instance.'"
                    """
                }
            }
        }
        stage('Check pg_basebackup Installation') {
            steps {
                sshagent(credentials: [env.SSH_KEY_CRED]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} "which pg_basebackup || sudo yum install -y postgresql*"
                    """
                }
            }
        }
        stage('Take PostgreSQL Snapshot') {
            steps {
                sshagent(credentials: [env.SSH_KEY_CRED]) {
                    script {
                        def backupDir = "/tmp/postgres_backup"
                        sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} \\
                        "pg_basebackup -D ${backupDir} -Ft -z -X fetch -v"
                        """
                        env.BACKUP_DIR = backupDir
                    }
                }
            }
        }
        stage('Verify Backup') {
            steps {
                sshagent(credentials: [env.SSH_KEY_CRED]) {
                    script {
                        sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} \\
                        "test -d ${env.BACKUP_DIR} && echo 'Backup verification successful.'"
                        """
                    }
                }
            }
        }
        stage('Upload Backup to S3') {
            steps {
                sshagent(credentials: [env.SSH_KEY_CRED]) {
                    script {
                        def backupFile = "postgres_backup_$(date +%Y%m%d%H%M%S).tar.gz"
                        sh """
                        ssh ec2-user@${EC2_HOST} \\
                        "aws s3 cp ${env.BACKUP_DIR}.tar.gz s3://${S3_BUCKET}/${backupFile} --region ${AWS_REGION}"
                        """
                        env.BACKUP_FILE = backupFile
                    }
                }
            }
        }
        stage('Verify S3 Upload') {
            steps {
                script {
                    def s3_check = sh(script: """
                    aws s3 ls s3://${S3_BUCKET}/${env.BACKUP_FILE} --region ${AWS_REGION}
                    """, returnStatus: true)
                    if (s3_check != 0) {
                        error "S3 upload verification failed."
                    }
                }
            }
        }
    }
    post {
        always {
            sshagent(credentials: [env.SSH_KEY_CRED]) {
                sh "ssh ec2-user@${EC2_HOST} 'rm -rf ${env.BACKUP_DIR}'"
            }
        }
    }
}
