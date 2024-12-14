def EC2_PUBLIC_IP = ""
def RDS_ENDPOINT = ""
def DEPLOYER_KEY_URI = ""

pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
        ECR_REPO_URL = '225989365312.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPO_NAME = 'enis-app'
        IMAGE_REPO = "${ECR_REPO_URL}/${ECR_REPO_NAME}"
        AWS_REGION = "us-east-1"
    }

    stages {
        stage('Provision Server and Database') {
            steps {
                script {
                    // Utilisation du conteneur Docker pour Terraform
                    dir('my-terraform-project/remote_backend') {
                        sh '''
                            docker run --rm \
                                -v $(pwd):/workspace \
                                -w /workspace \
                                hashicorp/terraform:latest terraform init
                        '''
                        // Apply Terraform configuration
                        sh '''
                            docker run --rm \
                                -v $(pwd):/workspace \
                                -w /workspace \
                                hashicorp/terraform:latest terraform apply --auto-approve
                        '''
                    }
                    dir('my-terraform-project') {
                        // Initialiser Terraform avec Docker
                        sh '''
                            docker run --rm \
                                -v $(pwd):/workspace \
                                -w /workspace \
                                hashicorp/terraform:latest terraform init
                        '''
                        sh '''
                            docker run --rm \
                                -v $(pwd):/workspace \
                                -w /workspace \
                                hashicorp/terraform:latest terraform plan -lock=false
                        '''
                        // Appliquer la configuration Terraform avec Docker
                        sh '''
                            docker run --rm \
                                -v $(pwd):/workspace \
                                -w /workspace \
                                hashicorp/terraform:latest terraform apply -lock=false --auto-approve
                        '''
                        // Récupérer l'IP publique de l'instance EC2
                        EC2_PUBLIC_IP = sh(
                            script: '''
                                docker run --rm \
                                    -v $(pwd):/workspace \
                                    -w /workspace \
                                    hashicorp/terraform:latest terraform output instance_details | grep "instance_public_ip" | awk '{print $3}' | tr -d '"'
                            ''',
                            returnStdout: true
                        ).trim()
                        // Récupérer l'endpoint RDS
                        RDS_ENDPOINT = sh(
                            script: '''
                                docker run --rm \
                                    -v $(pwd):/workspace \
                                    -w /workspace \
                                    hashicorp/terraform:latest terraform output rds_endpoint | grep "endpoint" | awk -F'=' '{print $2}' | tr -d '[:space:]"' | sed 's/:3306//'
                            ''',
                            returnStdout: true
                        ).trim()
                        DEPLOYER_KEY_URI = sh(
                            script: '''
                                docker run --rm \
                                    -v $(pwd):/workspace \
                                    -w /workspace \
                                    hashicorp/terraform:latest terraform output deployer_key_s3_uri | tr -d '"'
                            ''',
                            returnStdout: true
                        ).trim()
                        // Debugging: Print captured values
                        echo "EC2 Public IP: ${EC2_PUBLIC_IP}"
                        echo "RDS Endpoint: ${RDS_ENDPOINT}"
                        echo "Deployer Key URI: ${DEPLOYER_KEY_URI}"
                    }
                }
            }
        }

        stage('Update Frontend Configuration') {
            steps {
                script {
                    dir('enis-app-tp/frontend/src') {
                        writeFile file: 'config.js', text: """
                            export const API_BASE_URL = 'http://${EC2_PUBLIC_IP}:8000';
                        """
                        sh '''
                            echo "Contents of config.js after update:"
                            cat config.js
                        '''
                    }
                }
            }
        }

        stage('Update Backend Configuration') {
            steps {
                script {
                    dir('enis-app-tp/backend/backend') {
                        // Vérifier l'existence de settings.py
                        sh '''
                            if [ -f "settings.py" ]; then
                                echo "Found settings.py at $(pwd)"
                            else
                                echo "settings.py not found in $(pwd)!"
                                exit 1
                            fi
                        '''
                        // Mettre à jour le champ 'HOST' dans la section DATABASES
                        sh """
                            sed -i "/'HOST':/c\\ 'HOST': '${RDS_ENDPOINT}'," settings.py
                        """
                        // Vérifier la section DATABASES après la mise à jour
                        sh '''
                            echo "DATABASES section of settings.py after update:"
                            sed -n '/DATABASES = {/,/^}/p' settings.py
                        '''
                    }
                }
            }
        }
    }
}
