SETUP_NEEDED = false
pipeline {
    agent any
    environment {
        NODEJS_DIR = "/usr/local/lib/nodejs"
        N8N_SETUP_DIR = "${JENKINS_HOME}/workspace/n8n-setup"
        N8N_HOME = "${JENKINS_HOME}/workspace/n8n"
        PROXY_FILE = "/etc/apt/apt.conf.d/00-proxy"
        NODE_TAR_FILE = "node-v14.18.0-linux-x64.tar.gz"
        NODE_VER_BUILD = "node-v14.18.0-linux-x64"
    }
    stages {
        stage("Build N8N") {
            steps {
                dir("${JENKINS_HOME}/workspace") {
                    script {
                        if ( !fileExists (N8N_HOME) ) {
                            sh 'git clone https://github.com/krish918/n8n.git'
                            SETUP_NEEDED = true
                        }
                    }
                }
                dir( N8N_SETUP_DIR ) {
                    script {
                        if ( !fileExists ('./setup.conf') ) {

                            SETUP_NEEDED = true
                            sh 'which /usr/bin/npm'
                            if ( !fileExists( PROXY_FILE ) ) {
                                sh 'echo "Acquire::http::proxy \\"http://proxy-dmz.intel.com:911\\";\nAcquire::https::proxy \\"http://proxy-dmz.intel.com:912\\";" >> "$PROXY_FILE"'           
                            }
                            
                            sh 'apt-get update -y'
                            
                            if ( !fileExists ( NODE_TAR_FILE ) ) {
                                sh 'curl -O "https://nodejs.org/dist/v14.18.0/${NODE_TAR_FILE}"'
                            }
                            
                            sh "mkdir -p ${NODEJS_DIR}"
                            dir_empty = sh ( script: 'test -z "$(ls -A $NODEJS_DIR)"', returnStatus: true ) == 0
                            if ( dir_empty == true ) {
                                sh 'apt-get install -y tar gzip'
                                sh "tar -xvzf ${NODE_TAR_FILE} -C ${NODEJS_DIR}"
                            }

                            sh 'apt-get install -y build-essential python'

                            npm_exist = sh ( script: 'test -e /usr/bin/npm', returnStatus: true ) == 0
                            if ( npm_exist == false ) {
                                sh "ln -s /usr/bin/npm ${NODEJS_DIR}/${NODE_VER_BUILD}/bin/npm"
                            }

                            node_exist = sh ( script: 'test -e /usr/bin/npm', returnStatus: true ) == 0
                            if ( npm_exist == false ) {
                                sh "ln -s ${NODEJS_DIR}/${NODE_VER_BUILD}/bin/npm /usr/bin/npm"
                            }

                            lerna_exist = sh ( script: 'test -e /usr/bin/lerna', returnStatus: true ) == 0
                            if ( lerna_exist == false ) {
                                sh 'npm install -g lerna'
                                sh "ln -s ${NODEJS_DIR}/${NODE_VER_BUILD}/bin/lerna /usr/bin/lerna"
                            }
                        }
                    }
                }
                dir( N8N_HOME ) {
                    script {
                        UPDATED = false
                        if ( SETUP_NEEDED == false ) {
                            sh 'git remote update'
                            LOCAL = sh( script: 'git rev-parse @', returnStdout: true ).trim()
                            REMOTE = sh( script: 'git rev-parse @{u}', returnStdout: true ).trim()
                            if ( LOCAL != REMOTE ) {
                                sh 'git pull origin master'
                                UPDATED = true
                            }
                        }
                        if ( SETUP_NEEDED == true || UPDATED == true ) {
                            sh 'lerna bootstrap --hoist'
                            sh 'npm run build'
                            sh "$N8N_HOME/packages/cli/bin/n8n --version"

                            if ( !fileExists ("$N8N_SETUP_DIR/setup.conf")) {
                                sh 'echo "INITIAL_SETUP_DONE" >> "${N8N_SETUP_DIR}/setup.conf"'
                            }
                        }
                    }
                }
            }
        }
        
        stage("ImportCredentials") {
            steps {
                sh "$N8N_HOME/packages/cli/bin/n8n import:credentials --input=credentials.json"
                sh 'cp config "${JENKINS_HOME}/.n8n/"'
            }
        }
        stage("Execute") {
            steps {
                sh 'echo "Executing Workflow..."'
                sh "$N8N_HOME/packages/cli/bin/n8n execute --file workflow.json"
            }
        }
    }
}
