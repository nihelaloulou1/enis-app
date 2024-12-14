def EC2_PUBLIC_IP = ""
def RDS_ENDPOINT = ""
def DEPLOYER_KEY_URI = ""

pipeline {
    agent any  // Utilisation d'un agent générique pour toute la pipeline

    environment {
        AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
        ECR_REPO_URL = '225989365312.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPO_NAME = 'enis-app'
        IMAGE_REPO = "${ECR_REPO_URL}/${ECR_REPO_NAME}"
        AWS_REGION = "us-east-1"
        // Spécification d'un chemin absolu sous Windows
        WORKSPACE_PATH = 'C:/Jenkins/workspace/deploy_note_app'
    }

    stages {
        stage('Provision Server and Database') {
            steps {
                script {
                    // Utilisation de Docker pour exécuter Terraform
                    docker.image('hashicorp/terraform:latest').inside("-v ${WORKSPACE_PATH}:${WORKSPACE_PATH}") {
                        dir('my-terraform-project/remote_backend') {
                            sh "terraform init"
                            sh "terraform apply --auto-approve"
                        }

                        dir('my-terraform-project') {
                            sh "terraform init"
                            sh "terraform plan -lock=false"
                            sh "terraform apply -lock=false --auto-approve"

                            // Obtenir l'IP publique de l'EC2
                            EC2_PUBLIC_IP = sh(
                                script: 'terraform output instance_details | grep "instance_public_ip" | awk \'{print $3}\' | tr -d \'"\'',
                                returnStdout: true
                            ).trim()

                            // Obtenir l'endpoint RDS
                            RDS_ENDPOINT = sh(
                                script: '''
                                    terraform output rds_endpoint | grep "endpoint" | awk -F'=' '{print $2}' | tr -d '[:space:]"' | sed 's/:3306//'
                                ''',
                                returnStdout: true
                            ).trim()

                            // Obtenir l'URI de la clé de déploiement
                            DEPLOYER_KEY_URI = sh(
                                script: 'terraform output deployer_key_s3_uri | tr -d \'\"\'',
                                returnStdout: true
                            ).trim()

                            // Débogage : afficher les valeurs capturées
                            echo "EC2 Public IP: ${EC2_PUBLIC_IP}"
                            echo "RDS Endpoint: ${RDS_ENDPOINT}"
                            echo "Deployer Key URI: ${DEPLOYER_KEY_URI}"
                        }
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
                            echo "settings.py not found in $(pwd)! Exiting"
                            exit 1
                        fi
                        '''
                        
                        // Mettre à jour l'hôte dans la section DATABASES
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
