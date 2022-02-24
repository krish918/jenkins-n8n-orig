pipeline {
    agent any
    stages {
        stage("GetVersion") {
            steps {
                sh '/home/krisk918/n8n/packages/cli/bin/n8n --version'
            }
        }
        stage("ImportCredentials") {
            steps {
                sh '/home/krisk918/n8n/packages/cli/bin/n8n import:credentials --input=credentials.json'
                sh 'cp config /var/lib/jenkins/.n8n/'
            }
        }
        stage("Execute") {
            steps {
                sh 'echo "Executing Workflow..."'
                sh '/home/krisk918/n8n/packages/cli/bin/n8n execute --file workflow.json'
            }
        }
    }
}