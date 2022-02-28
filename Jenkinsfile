CLONE = false
pipeline {
    agent any
    stages {
        stage("Build N8N") {
            steps {
                dir("${JENKINS_HOME}/workspace") {
                    script {
                        result = sh( script: 'test -d ./n8n', returnStatus: true ) == 0
                        if ( result == false ) {
                            sh 'git clone https://github.com/krish918/n8n.git'
                            CLONE = true
                        }
                    }
                }
                dir("${JENKINS_HOME}/workspace") {
                    script {
                        node_res = sh( script: 'test -e ./node-v14.18.0-linux-x64.tar.gz', returnStatus: true ) == 0
                        if ( node_res == false ) {
                            sh 'curl -O https://nodejs.org/dist/v14.18.0/node-v14.18.0-linux-x64.tar.gz'
                        }
                        sh "mkdir -p ${JENKINS_HOME}/workspace/nodejs"
                        dir_empty = sh( script: 'test -z "$(ls -A "$JENKINS_HOME"/workspace/nodejs)"', returnStatus: true ) == 0
                        if ( dir_empty == true ) {
                            sh 'apt-get update -y'
                            sh 'apt-get install -y tar'
                            sh 'apt-get install -y gzip'
                            sh "tar -xvzf node-v14.18.0-linux-x64.tar.gz -C ${JENKINS_HOME}/workspace/nodejs"
                        }
                        sh 'apt-get install -y build-essential python'
                        npm_res = sh( script: 'test -f /usr/bin/npm', returnStatus: true ) == 0
                        if ( npm_res == false ) {
                            sh "ln -s ${JENKINS_HOME}/workspace/nodejs/node-v14.18.0-linux-x64/bin/npm /usr/bin/npm"
                            sh "ln -s ${JENKINS_HOME}/workspace/nodejs/node-v14.18.0-linux-x64/bin/node /usr/bin/node"
                        }
                        lerna_res = sh( script: 'test -f /usr/bin/lerna', returnStatus: true ) == 0
                        if ( result == false ) {
                            sh 'npm install -g lerna'
                            sh "ln -s ${JENKINS_HOME}/workspace/nodejs/node-v14.18.0-linux-x64/bin/lerna /usr/bin/lerna"
                        }
                    }
                }
                dir("${JENKINS_HOME}/workspace/n8n") {
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
                            sh 'npm run build'
                            sh './packages/cli/bin/n8n --version'
                        }
                    }
                }
            }
        }
        
        stage("ImportCredentials") {
            steps {
                sh "${JENKINS_HOME}/workspace/n8n/packages/cli/bin/n8n import:credentials --input=credentials.json"
                sh 'cp config ${JENKINS_HOME}/.n8n/'
            }
        }
        stage("Execute") {
            steps {
                sh 'echo "Executing Workflow..."'
                sh "${JENKINS_HOME}/workspace/n8n/packages/cli/bin/n8n execute --file workflow.json"
            }
        }
    }
}