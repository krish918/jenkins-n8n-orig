pipeline {
    agent any
    stages {
        stage("FetchN8N") {
            steps {
                dir('/var/lib/jenkins/workspace') {
                    script {
                        result = sh(script: 'test -d ./n8n', returnStatus: true) == 0
                        if( result == 0) {
                            sh 'git clone https://github.com/krish918/n8n.git'
                        }
                    }
                }
            }
        }
        stage("InstallBuildTools") {
            steps {
                dir('/var/lib/jenkins/workspace/n8n') {
                    sh 'sudo apt-get install -y build-essential python'
                    sh 'sudo npm install -g lerna'
                }
            }
        }
        stage("BuildN8N") {
            steps {
                dir('/var/lib/jenkins/workspace/n8n') {
                    sh 'lerna bootstrap --hoist'
                    sh 'sudo npm run build'
                }
            }
        }
        stage("CheckVersion") {
            steps {
                dir('/var/lib/jenkins/workspace/n8n') {
                    sh './packages/cli/bin/n8n --version'
                }
            }
        }
        stage("ImportCredentials") {
            steps {
                dir('/var/lib/jenkins/workspace/n8n') {
                    sh './packages/cli/bin/n8n import:credentials --input=credentials.json'
                    sh 'cp config /var/lib/jenkins/.n8n/'
                }
            }
        }
        stage("Execute") {
            steps {
                dir('/var/lib/jenkins/workspace/n8n') {
                    sh 'echo "Executing Workflow..."'
                    sh './packages/cli/bin/n8n execute --file workflow.json'
                }
            }
        }
    }
}