BUILD_NEEDED = false
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
        NODE_DIST = "https://nodejs.org/dist/v14.18.0/${NODE_TAR_FILE}"

        N8N_SETUP_DIR = "${HOME}/workspace/n8n-setup"
        N8N_HOME = "${HOME}/workspace/n8n"

        //PROXY_FILE = "/etc/apt/apt.conf.d/00-proxy"
        //VERIFY_PEER_CONFIG_FILE = "/etc/apt/apt.conf.d/99-verify-peer"

        __REPO_N8N = "https://github.com/krish918/n8n.git"
    }
    stages {
        stage("Build N8N") {
            steps {
                
                // We will clone and setup N8N one level above the current working dir
                
                dir("${HOME}/workspace") {
                    script {
                         
                        // check if N8N directory in the workspace already exists, otherwise clone it
                        
                        if ( !fileExists (N8N_HOME) ) {
                            sh 'git clone "$__REPO_N8N"'
                            
                            // if it is cloned, then it needs to be built
                            BUILD_NEEDED = true
                        }
                    }
                }
                dir( N8N_SETUP_DIR ) {
                    script {
                        /*
                        setup.conf is created in N8N_SETUP_ DIR once the BUILD is successful. 
                        If it is already available, then we don't need to RE-BUILD. 
                        */
                        if ( !fileExists ('./setup.conf') ) {

                            BUILD_NEEDED = true
                            
                            // commenting out as we don't need to setup proxy on aws node.
                            /*
                            if ( !fileExists( PROXY_FILE ) ) {
                                sh 'echo "Acquire::http::proxy \\"http://proxy-dmz.intel.com:911\\";\nAcquire::https::proxy \\"http://proxy-dmz.intel.com:912\\";" >> "$PROXY_FILE"'           
                            }
                            */
                            
                            sh 'sudo apt-get update -y'
                            
                            // if node tarball is not available, then download it.
                            
                            if ( !fileExists ( NODE_TAR_FILE ) ) {
                                sh 'sudo apt-get install -y curl'
                                sh 'curl -O "${NODE_DIST}"'
                            }
                            
                            // we make the dir where we will be installing N8N recommended ver of node, in case node is not pre-installed.
                            
                            sh "sudo mkdir -p ${NODEJS_DIR}"
                            
                            // extract the node tarball, if it is not already extracted
                            
                            dir_empty = sh ( script: 'test -z "$(ls -A $NODEJS_DIR)"', returnStatus: true ) == 0
                            if ( dir_empty == true ) {
                                sh 'sudo apt-get install -y tar gzip'
                                sh "sudo tar -xzf ${NODE_TAR_FILE} -C ${NODEJS_DIR}"
                                sh 'sudo chown -R $(whoami) "$NODEJS_DIR"' 
                            }

                            // install build tools to be able to build N8N
                            
                            sh 'sudo apt-get install -y build-essential python'

                            /***
                                If npm and node are not already available, 
                                make them available via a symbolic link to the node version, which we just 
                                extracted in in NODEJS_DIR
                             ***/
                            npm_exist = sh ( script: 'test -L /usr/bin/npm', returnStatus: true ) == 0
                            if ( npm_exist == false ) {
                                sh "sudo ln -s ${NODEJS_DIR}/${NODE_VER_BUILD}/bin/npm /usr/bin/npm"
                            }

                            node_exist = sh ( script: 'test -L /usr/bin/node', returnStatus: true ) == 0
                            if ( npm_exist == false ) {
                                sh "sudo ln -s ${NODEJS_DIR}/${NODE_VER_BUILD}/bin/node /usr/bin/node"
                            }
                            
                            // check for existence of lerna (needed to build N8N) and install it if not available
                            
                            lerna_exist = sh ( script: 'test -L /usr/bin/lerna', returnStatus: true ) == 0
                            if ( lerna_exist == false ) {
                                sh 'sudo npm install -g lerna'
                                //sh "sudo ln -s ${NODEJS_DIR}/${NODE_VER_BUILD}/bin/lerna /usr/bin/lerna"
                            }
                        }
                    }
                }
                dir( N8N_HOME ) {
                    script {
                        UPDATED = false
                        /*
                        If N8N is already built, then check if there are new commits on the repo.
                        If yes, the turn on the UPDATED flag, so that N8N can be re-built.
                        */
                        if ( BUILD_NEEDED == false ) {
                            sh 'git remote update'
                            LOCAL = sh( script: 'git rev-parse @', returnStdout: true ).trim()
                            REMOTE = sh( script: 'git rev-parse @{u}', returnStdout: true ).trim()
                            if ( LOCAL != REMOTE ) {
                                sh 'git pull origin master'
                                UPDATED = true
                            }
                        }
                        
                        /*
                        Build N8N, either it is first time setup (BUILD_NEEDED = true) or 
                        if there are some new commits on the repo (UPDATED = true)
                        */
                        
                        if ( BUILD_NEEDED == true || UPDATED == true ) {
                            sh 'lerna bootstrap --hoist'
                            sh 'sudo npm run build'
                            sh "$N8N_HOME/packages/cli/bin/n8n --version"
                            
                            // create a dummy file in N8N_SETUP_DIR which signals that N8N is built succesfully.
                            
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
                    
                    /*
                    check whether docker exists, otherwise setup docker on the current node.
                    */
                    docker_exist = sh (script : 'command -v docker', returnStatus : true) == 0
                    if ( !docker_exist ) {
                        
                        sh 'sudo apt-get install -y ca-certificates gnupg lsb-release'

                        //sh 'echo "Acquire { https::Verify-Peer \\"false\\" }" >> "$VERIFY_PEER_CONFIG_FILE"'

                        sh 'curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --batch --yes --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg'
                        sh 'echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null'
                        sh 'sudo apt-get update -y'
                        sh 'sudo apt-get install -y docker-ce docker-ce-cli containerd.io'
                        
                        // check if group docker exist, otherwise create it.
                        docker_grp = sh (script : 'sudo getent group docker', returnStatus : true ) == 0
                        if ( !docker_grp ) {
                            sh 'sudo groupadd docker'   
                        }
                        
                        // add current user to docker group to be able to run docker without sudo
                        sh 'sudo usermod -aG docker "$(whoami)"'
                    }
                    
                    sh 'sudo systemctl start docker'

                    /*
                        Install docker-compose if not available on current node.
                    */
                    docker_compose = sh (script : 'command -v docker-compose', returnStatus : true) == 0
                    if ( !docker_compose ) {
                        sh 'sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose'
                        sh 'sudo chmod +x /usr/local/bin/docker-compose'
                        sh 'sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose'
                    }

                    // if the docker-compose file exists in the current repo, spin-up the services.
                    
                    if ( fileExists ('docker-compose.yml') ) {
                        sh 'sudo docker-compose up -d'
                    }
                }
            }
        }
        stage("Import & Execute Workflow") {
            steps {
                
                // import credentials from this repo into N8N
                
                sh "$N8N_HOME/packages/cli/bin/n8n import:credentials --input=credentials.json"
                
                // copy the encryption key (needed to decrypt credentials) into .n8n directory of current user's home. 
                
                sh 'cp config "${HOME}/.n8n/"'
                
                // execute the workflow
                
                sh 'echo "Executing Workflow..."'
                sh "$N8N_HOME/packages/cli/bin/n8n execute --file workflow.json"
            }
        }
    }
}
