SETUP_NEEDED = false
pipeline {
    agent {
        node {
            label 'aws-ec2'
        }
    }
    environment {
        NODEJS_DIR = "/usr/local/lib/nodejs"
        NODE_TAR_FILE = "node-v14.18.0-linux-x64.tar.gz"
        NODE_VER_BUILD = "node-v14.18.0-linux-x64"

        N8N_SETUP_DIR = "${HOME}/workspace/n8n-setup"
        N8N_HOME = "${HOME}/workspace/n8n"

        DL_STREAMER_DIR = "${N8N_SETUP_DIR}/dl-streamer-setup"

        //PROXY_FILE = "/etc/apt/apt.conf.d/00-proxy"
        //VERIFY_PEER_CONFIG_FILE = "/etc/apt/apt.conf.d/99-verify-peer"

        __REPO_N8N = "https://github.com/krish918/n8n.git"
        __REPO_MICROSERVICE = "https://github.com/krish918/dl-streamer-setup.git"
    }
    stages {
        stage("Build N8N") {
            steps {
                dir("${HOME}/workspace") {
                    script {
                        if ( !fileExists (N8N_HOME) ) {
                            sh 'git clone "$__REPO_N8N"'
                            SETUP_NEEDED = true
                        }
                    }
                }
                dir( N8N_SETUP_DIR ) {
                    script {
                        if ( !fileExists ('./setup.conf') ) {

                            SETUP_NEEDED = true
                            /*
                            if ( !fileExists( PROXY_FILE ) ) {
                                sh 'echo "Acquire::http::proxy \\"http://proxy-dmz.intel.com:911\\";\nAcquire::https::proxy \\"http://proxy-dmz.intel.com:912\\";" >> "$PROXY_FILE"'           
                            }
                            */
                            
                            sh 'sudo apt-get update -y'
                            
                            if ( !fileExists ( NODE_TAR_FILE ) ) {
                                sh 'sudo apt-get install -y curl'
                                sh 'curl -O "https://nodejs.org/dist/v14.18.0/${NODE_TAR_FILE}"'
                            }
                            
                            sh "sudo mkdir -p ${NODEJS_DIR}"
                            dir_empty = sh ( script: 'test -z "$(ls -A $NODEJS_DIR)"', returnStatus: true ) == 0
                            if ( dir_empty == true ) {
                                sh 'sudo apt-get install -y tar gzip'
                                sh "sudo tar -xvzf ${NODE_TAR_FILE} -C ${NODEJS_DIR}"
                                sh 'sudo chown -R $(whoami) "$NODEJS_DIR"' 
                            }

                            sh 'sudo apt-get install -y build-essential python'

                            npm_exist = sh ( script: 'test -L /usr/bin/npm', returnStatus: true ) == 0
                            if ( npm_exist == false ) {
                                sh "sudo ln -s ${NODEJS_DIR}/${NODE_VER_BUILD}/bin/npm /usr/bin/npm"
                            }

                            node_exist = sh ( script: 'test -L /usr/bin/node', returnStatus: true ) == 0
                            if ( npm_exist == false ) {
                                sh "sudo ln -s ${NODEJS_DIR}/${NODE_VER_BUILD}/bin/node /usr/bin/node"
                            }

                            lerna_exist = sh ( script: 'test -L /usr/bin/lerna', returnStatus: true ) == 0
                            if ( lerna_exist == false ) {
                                sh 'sudo npm install -g lerna'
                                sh "sudo ln -s ${NODEJS_DIR}/${NODE_VER_BUILD}/bin/lerna /usr/bin/lerna"
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
                            sh 'sudo npm run build'
                            sh "$N8N_HOME/packages/cli/bin/n8n --version"

                            if ( !fileExists ("$N8N_SETUP_DIR/setup.conf")) {
                                sh 'echo "INITIAL_SETUP_DONE" >> "${N8N_SETUP_DIR}/setup.conf"'
                            }
                        }
                    }
                }
            }
        }
        
        stage("Setup Microservices") {
            steps {
                script {
                    docker_exist = sh (script : 'command -v docker', returnStatus : true) == 0
                    if ( !docker_exist ) {
                        
                        sh 'sudo apt-get install -y ca-certificates gnupg lsb-release'

                        //sh 'echo "Acquire { https::Verify-Peer \\"false\\" }" >> "$VERIFY_PEER_CONFIG_FILE"'

                        sh 'curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --batch --yes --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg'
                        sh 'echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null'
                        sh 'sudo apt-get update -y'
                        sh 'sudo apt-get install -y docker-ce docker-ce-cli containerd.io'
                        
                        docker_grp = sh (script : 'sudo getent group docker', returnStatus : true ) == 0
                        if ( !docker_grp ) {
                            sh 'sudo groupadd docker'   
                        }
                        sh 'sudo usermod -aG docker "$(whoami)"'
                    }
                    
                    sh 'service docker start'

                    docker_compose = sh (script : 'command -v docker-compose', returnStatus : true) == 0
                    if ( !docker_compose ) {
                        sh 'sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose'
                        sh 'sudo chmod +x /usr/local/bin/docker-compose'
                        sh 'sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose'
                    }

                    if ( !fileExists (DL_STREAMER_DIR) ) {
                        dir ( N8N_SETUP_DIR ) {
                            sh 'git clone "$__REPO_MICROSERVICE"'
                        }
                    }
                        
                    dir ( DL_STREAMER_DIR ) {
                        sh 'sudo docker-compose up -d'
                    }
                }
            }
        }
        stage("Import & Execute Workflow") {
            steps {
                sh "$N8N_HOME/packages/cli/bin/n8n import:credentials --input=credentials.json"
                sh 'cp config "${HOME}/.n8n/"'
                sh 'echo "Executing Workflow..."'
                sh "$N8N_HOME/packages/cli/bin/n8n execute --file workflow.json"
            }
        }
    }
}
