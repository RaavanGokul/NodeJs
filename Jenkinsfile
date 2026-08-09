pipeline {

    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }

    triggers {
        githubPush()
    }

    environment {
        NODEJS_SERVER_IP = "172.31.4.119"
        NODEJS_SERVER_USER = "ec2-user"
        REMOTE_PATH = "/home/ec2-user/nodejs-app"
    }

    tools {
        nodejs "NodeJs-26.7"
    }

    stages {

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "===== Node.js Version ====="
                    node -v

                    echo "===== NPM Version ====="
                    npm -v

                    echo "===== Node Path ====="
                    which node

                    echo "===== NPM Path ====="
                    which npm

                    echo "===== Installing Dependencies ====="

                    if [ -f package-lock.json ]; then
                        npm ci
                    else
                        npm install
                    fi
                '''
            }
        }

        stage('Copy Files to Remote Server') {
            steps {
                sshagent(['npm-server']) {
                    sh '''
                        echo "===== Creating Remote Directory ====="

                        ssh -o StrictHostKeyChecking=no \
                            $NODEJS_SERVER_USER@$NODEJS_SERVER_IP \
                            "mkdir -p $REMOTE_PATH"

                        echo "===== Copying Application ====="

                        rsync -avz \
                            --exclude=node_modules \
                            --exclude=.git \
                            ./ \
                            $NODEJS_SERVER_USER@$NODEJS_SERVER_IP:$REMOTE_PATH/
                    '''
                }
            }
        }

        stage('Start Node.js Application') {
            steps {
                sshagent(['npm-server']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                            $NODEJS_SERVER_USER@$NODEJS_SERVER_IP "
                            
                            set -e

                            cd $REMOTE_PATH

                            echo '===== Remote Node.js Version ====='
                            node -v

                            echo '===== Remote NPM Version ====='
                            npm -v

                            echo '===== Installing Production Dependencies ====='

                            if [ -f package-lock.json ]; then
                                npm ci --omit=dev
                            else
                                npm install --omit=dev
                            fi

                            echo '===== Checking PM2 ====='
                            pm2 -v

                            echo '===== Starting/Restarting Application ====='

                            if pm2 describe my-app > /dev/null 2>&1; then
                                pm2 restart my-app --update-env
                            else
                                pm2 start app.js --name my-app
                            fi

                            pm2 save
                        "
                    '''
                }
            }
        }
    }

    post {

        always {

            script {

                def buildStatus = currentBuild.currentResult

                emailext(
                    body: """
                    <!DOCTYPE html>
                    <html>
                    <head>
                        <style>
                            body {
                                font-family: Arial, sans-serif;
                            }

                            .header {
                                background-color: #f4f5f7;
                                padding: 10px;
                                border-bottom: 2px solid #ccc;
                            }

                            .success {
                                color: green;
                                font-weight: bold;
                            }

                            .failure {
                                color: red;
                                font-weight: bold;
                            }

                            .content {
                                margin-top: 15px;
                            }
                        </style>
                    </head>

                    <body>

                        <div class="header">
                            <h2>Jenkins Build Report</h2>
                        </div>

                        <div class="content">

                            <p>
                                <strong>Job:</strong>
                                ${env.JOB_NAME}
                            </p>

                            <p>
                                <strong>Build Number:</strong>
                                #${env.BUILD_NUMBER}
                            </p>

                            <p>
                                <strong>Status:</strong>
                                <span class="${buildStatus == 'SUCCESS' ? 'success' : 'failure'}">
                                    ${buildStatus}
                                </span>
                            </p>

                            <p>
                                Check the full console output
                                <a href="${env.BUILD_URL}">here</a>.
                            </p>

                        </div>

                    </body>
                    </html>
                    """,

                    mimeType: 'text/html',

                    subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ${buildStatus}",

                    to: 'gokulkrishnan3123@gmail.com'
                )
            }
        }
    }
}