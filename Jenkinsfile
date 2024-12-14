pipeline {
    agent any

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    checkout scmGit(
                        branches: [[name: '*/main']],  
                        extensions: [], 
                        userRemoteConfigs: [[
                            credentialsId: 'githubtoken',  
                            url: 'https://github.com/nihelaloulou1/enis-app'  
                        ]]
                    )
                }
            }
        }
    }
}
