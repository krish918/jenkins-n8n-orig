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
                dir("${JENKINS_HOME}/workspace/n8n-setup") {
                    script {
                        if ( !fileExist ('./setup.conf')) {

                            if ( !fileExists( '/etc/apt/apt.conf.d/00-proxy' ) ) {
                                sh 'echo "Acquire::http::proxy \\"http://proxy-dmz.intel.com:911\\";\nAcquire::https::proxy \\"http://proxy-dmz.intel.com:912\\";" >> /etc/apt/apt.conf.d/00-proxy'           
                            }
                            
                            sh 'apt-get update -y'
                            
                            if ( !fileExists ('./node-v14.18.0-linux-x64.tar.gz') ) {
                                sh 'curl -O https://nodejs.org/dist/v14.18.0/node-v14.18.0-linux-x64.tar.gz'
                            }
                            
                            sh "mkdir -p /usr/local/lib/nodejs"
                            dir_empty = sh( script: 'test -z "$(ls -A /usr/local/lib/nodejs)"', returnStatus: true ) == 0
                            if ( dir_empty == true ) {
                                sh 'apt-get install -y tar'
                                sh 'apt-get install -y gzip'
                                sh "tar -xvzf node-v14.18.0-linux-x64.tar.gz -C /usr/local/lib/nodejs"
                            }

                            sh 'apt-get install -y build-essential python'

                            if ( !fileExists ('/usr/bin/npm') ) {
                                sh "ln -s /usr/local/lib/nodejs/nodejs/node-v14.18.0-linux-x64/bin/npm /usr/bin/npm"
                                sh "ln -s /usr/local/lib/nodejs/node-v14.18.0-linux-x64/bin/node /usr/bin/node"
                            }
                            
                            if ( !fileExists ('/usr/bin/lerna') ) {
                                sh 'npm install -g lerna'
                                sh "ln -s /usr/local/lib/nodejs/node-v14.18.0-linux-x64/bin/lerna /usr/bin/lerna"
                            }
                        }
                    }
                }
                dir("${JENKINS_HOME}/workspace/n8n") {
                    script {
                        UPDATED = false
                        if ( CLONE == false ) {
                            sh 'git remote update'
                            LOCAL = sh( script: 'git rev-parse @', returnStdout: true ).trim()
                            REMOTE = sh( script: 'git rev-parse @{u}', returnStdout: true ).trim()
                            if ( LOCAL != REMOTE ) {
                                sh 'git pull origin master'
                                UPDATED = true
                            }
                        }
                        if ( CLONE == true || UPDATED == true ) {
                            sh 'lerna bootstrap --hoist'
                            sh 'npm run build'
                            sh './packages/cli/bin/n8n --version'

                            if ( !fileExist ("${JENKINS_HOME}/workspace/n8n-setup/setup.conf")) {
                                sh 'echo "DONE" >> ${JENKINS_HOME}/workspace/n8n-setup/setup.conf'
                            }
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
