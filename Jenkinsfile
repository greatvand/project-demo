pipeline {
    agent any

    environment {
        TF_IN_AUTOMATION = 'true'
        TF_CLI_ARGS = '-no-color'
        SSH_CRED_ID = 'ec2-ssh-key'
        TF_CLI_CONFIG_FILE = credentials('aws-credentials')
    }

    stages {

        stage('Terraform Provisioning') {
            steps {
                script {
                    sh 'terraform init'
                    sh 'terraform apply -auto-approve'

                    // 1. Extract Public IP Address
                    env.INSTANCE_IP = sh(
                        script: 'terraform output -raw instance_public_ip',
                        returnStdout: true
                    ).trim()

                    // 2. Extract Instance ID
                    env.INSTANCE_ID = sh(
                        script: 'terraform output -raw instance_id',
                        returnStdout: true
                    ).trim()

                    echo "Provisioned Instance IP: ${env.INSTANCE_IP}"
                    echo "Provisioned Instance ID: ${env.INSTANCE_ID}"

                    // 3. Dynamic inventory
                    sh "echo '${env.INSTANCE_IP}' > dynamic_inventory.ini"
                }
            }
        }

        stage('Wait for AWS Instance Status') {
            steps {
                echo "Waiting for instance ${env.INSTANCE_ID} to pass AWS health checks..."

                sh "aws ec2 wait instance-status-ok --instance-ids ${env.INSTANCE_ID} --region us-east-1"

                echo 'AWS instance health checks passed. Proceeding to Ansible.'
            }
        }

        stage('Ansible Configuration') {
            steps {
                ansiblePlaybook(
                    playbook: 'playbooks/grafana.yml',
                    inventory: 'dynamic_inventory.ini',
                    credentialsId: SSH_CRED_ID
                )
            }
        }

        stage('Validate Destroy') {
            input {
                message "Do you want to destroy?"
                ok "Destroy"
            }
            steps {
                echo "Destroy Approved"
            }
        }
    }

    post {
        always {
            sh 'rm -f dynamic_inventory.ini'
        }

        success {
            echo "success"
        }

        failure {
            sh 'terraform destroy -auto-approve'
        }
    }
}
