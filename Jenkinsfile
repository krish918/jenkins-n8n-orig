pipeline {
    agent any
    stages {
        stage("InstallN8N") {
            steps {
                sh 'echo "intel.1234" | sudo -S npm install n8n -g'
            }
        }
        stage("CheckVersion") {
            steps {
                sh 'n8n --version'
            }
        }
        stage("ImportCredentials") {
            steps {
                sh 'n8n import:credentials --input=credentials.json'
                sh 'cp config /var/lib/jenkins/.n8n/'
            }
        }
        stage("Execute") {
            steps {
                sh 'echo "Executing Workflow..."'
                sh 'n8n execute --file workflow.json'
            }
        }
    }
}