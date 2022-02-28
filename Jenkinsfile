CLONE = false
pipeline {
    agent any
    stages {
        stage("Build N8N") {
            steps {
                dir('/var/lib/jenkins/workspace') {
                    script {
                        result = sh( script: 'test -d ./n8n', returnStatus: true ) == 0
                        if ( result == false ) {
                            sh 'git clone https://github.com/krish918/n8n.git'
                            CLONE = true
                        }
                    }
                }
                dir('/var/lib/jenkins/workspace') {
                    script {
                        node_res = sh( script: 'test -e ./node-v14.18.0-linux-x64.tar.xz', returnStatus: true ) == 0
                        if ( node_res == false ) {
                            sh 'wget https://nodejs.org/dist/v14.18.0/node-v14.18.0-linux-x64.tar.xz'
                        }
                        sh 'sudo mkdir -p /usr/local/lib/nodejs'
                        dir_empty = sh( script: 'test -z $(ls -A /usr/local/lib/nodejs)', returnStatus: true ) == 0
                        if ( dir_empty == true ) {
                            sh 'sudo tar -xJvf node-v14.18.0-linux-x64.tar.xz -C /usr/local/lib/nodejs'
                        }
                        sh 'sudo apt-get install -y build-essential python'
                        npm_res = sh( script: 'test -f /usr/bin/npm', returnStatus: true ) == 0
                        if ( npm_res == false ) {
                            sh 'sudo ln -s /usr/local/lib/nodejs/node-v14.18.0-linux-x64/bin/npm /usr/bin/npm'
                            sh 'sudo ln -s /usr/local/lib/nodejs/node-v14.18.0-linux-x64/bin/node /usr/bin/node'
                        }
            
                        lerna_res = sh( script: 'test -f /usr/bin/lerna', returnStatus: true ) == 0
                        if ( result == false ) {
                            sh 'sudo npm install -g lerna'
                            sh 'sudo ln -s /usr/local/lib/nodejs/node-v14.18.0-linux-x64/bin/lerna /usr/bin/lerna'
                        }
                    }
                }
                dir('/var/lib/jenkins/workspace/n8n') {
                    script {
                        UPDATED = false
                        if ( CLONE == false ) {
                            sh 'git remote update'
                            LOCAL = sh( script: 'git rev-parse @', returnStdout: true )
                            REMOTE = sh( script: 'git rev-parse @{u}', returnStdout: true )
                            if ( LOCAL != REMOTE ) {
                                sh 'git pull origin master'
                                UPDATED = true
                            }
                        }
                        if ( CLONE == true || UPDATED == true ) {
                            sh 'lerna bootstrap --hoist'
                            sh 'sudo npm run build'
                            sh './packages/cli/bin/n8n --version'
                        }
                    }
                }
            }
        }
        
        stage("ImportCredentials") {
            steps {
                sh '/var/lib/jenkins/workspace/n8n/packages/cli/bin/n8n import:credentials --input=credentials.json'
                sh 'cp config /var/lib/jenkins/.n8n/'
            }
        }
        stage("Execute") {
            steps {
                dir('/var/lib/jenkins/workspace/n8n') {
                    sh 'echo "Executing Workflow..."'
                    sh '/var/lib/jenkins/workspace/n8n/packages/cli/bin/n8n execute --file workflow.json'
                }
            }
        }
    }
}