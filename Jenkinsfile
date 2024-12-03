pipeline {
    agent any
    parameters {
        string(name: 'AWS_REGION', defaultValue: '', description: 'Region of the S3 bucket')
        string(name: 'EC2_REGION', defaultValue: '', description: 'Region of the EC2 instance')
        string(name: 'S3_BUCKET', defaultValue: '', description: 'Name of the S3 bucket')
        string(name: 'EC2_HOST', defaultValue: '', description: 'EC2 instance hostname or IP')
        string(name: 'NETWORK', defaultValue: '', description: 'Name of Instance we are taking the snapshot from')
        credentials(name: 'SSH_KEY_CRED', credentialType: 'SSHUserPrivateKey', description: 'Jenkins credentials for SSH')
    }
    stages {
        stage('SSH to EC2 Instance') {
            steps {
                sshagent(credentials: [env.SSH_KEY_CRED]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} "echo 'Connected to EC2 instance.'"
                    """
                }
            }
        }
        stage('Check pg_basebackup Installation') {
            steps {
                sshagent(credentials: [env.SSH_KEY_CRED]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} \\
                    "which pg_basebackup || sudo apt-get update && sudo apt-get install -y postgresql-client"
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
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} \\
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
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} \\
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
                        def backupFile = sh(script: "echo postgres_backup_\$(date +%Y)", returnStdout: true).trim()
                        sh """
                        ssh ubuntu@${EC2_HOST} \\
                        "aws s3 cp ${env.BACKUP_DIR}.tar.gz s3://${S3_BUCKET}/${NETWORK}/${backupFile} --region ${AWS_REGION}"
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
                    aws s3 ls s3://${S3_BUCKET}/${NETWORK}/${backupFile} --region ${AWS_REGION}
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
                sh "ssh ubuntu@${EC2_HOST} 'rm -rf ${env.BACKUP_DIR}'"
            }
        }
    }
}
