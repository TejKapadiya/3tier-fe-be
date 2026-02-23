pipeline {
    agent any

    environment {
        REPO_URL    = 'https://github.com/TejKapadiya/3tier-fe-be.git'
        FE_DIR      = '/home/ubuntu/3tier-fe-be/frontend'
        BE_DIR      = '/home/ubuntu/3tier-fe-be/nodejs-express-mysql'
        FE_PM2      = 'vite-frontend'
        BE_PM2      = 'node-app'
    }

    options {
        // Keep only last 10 builds to save disk space
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Fail if pipeline runs longer than 20 minutes
        timeout(time: 20, unit: 'MINUTES')
    }

    stages {

        // ─────────────────────────────────────────
        stage('Checkout') {
        // ─────────────────────────────────────────
            steps {
                echo "Cloning branch: main from ${REPO_URL}"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: "${REPO_URL}"]]
                ])
            }
        }

        // ─────────────────────────────────────────
        stage('Install Backend Dependencies') {
        // ─────────────────────────────────────────
            steps {
                echo "Installing Node.js dependencies for backend..."
                dir("${BE_DIR}") {
                    sh 'npm install --production'
                }
            }
        }

        // ─────────────────────────────────────────
        stage('Install Frontend Dependencies') {
        // ─────────────────────────────────────────
            steps {
                echo "Installing Node.js dependencies for frontend..."
                dir("${FE_DIR}") {
                    sh 'npm install'
                }
            }
        }

        // ─────────────────────────────────────────
        stage('Build Frontend (Nuxt SSR)') {
        // ─────────────────────────────────────────
            steps {
                echo "Building Nuxt frontend in SSR mode..."
                dir("${FE_DIR}") {
                    sh 'npm run build'
                }
            }
        }

        // ─────────────────────────────────────────
        stage('Restart Backend via PM2') {
        // ─────────────────────────────────────────
            steps {
                echo "Restarting backend PM2 process: ${BE_PM2}"
                sh '''
                    # If process exists restart it, otherwise start fresh
                    if sudo -u ubuntu pm2 describe ${BE_PM2} > /dev/null 2>&1; then
                        sudo -u ubuntu pm2 restart ${BE_PM2}
                    else
                        echo "PM2 process ${BE_PM2} not found, starting it..."
                        sudo -u ubuntu pm2 start ${BE_DIR}/server.js --name ${BE_PM2}
                    fi
                '''
            }
        }

        // ─────────────────────────────────────────
        stage('Restart Frontend via PM2') {
        // ─────────────────────────────────────────
            steps {
                echo "Restarting frontend PM2 process: ${FE_PM2}"
                sh '''
                    if sudo -u ubuntu pm2 describe ${FE_PM2} > /dev/null 2>&1; then
                        sudo -u ubuntu pm2 restart ${FE_PM2}
                    else
                        echo "PM2 process ${FE_PM2} not found, starting it..."
                        sudo -u ubuntu pm2 start "npm run start" --name ${FE_PM2} --cwd ${FE_DIR}
                    fi
                '''
            }
        }

        // ─────────────────────────────────────────
        stage('Reload Nginx') {
        // ─────────────────────────────────────────
            steps {
                echo "Reloading Nginx to pick up any config changes..."
                sh 'sudo systemctl reload nginx'
            }
        }

        // ─────────────────────────────────────────
        stage('Health Check') {
        // ─────────────────────────────────────────
            steps {
                echo "Running PM2 status check..."
                sh 'sudo -u ubuntu pm2 list'
            }
        }

    } // end stages

    post {
        success {
            echo """
            ================================================
             DEPLOYMENT SUCCESSFUL
             Branch : main
             Frontend PM2 : ${FE_PM2}
             Backend PM2  : ${BE_PM2}
            ================================================
            """
        }
        failure {
            echo """
            ================================================
             DEPLOYMENT FAILED
             Check the stage logs above for details.
             Tip: Run 'pm2 logs' on the server for app errors.
            ================================================
            """
        }
        always {
            // Clean Jenkins workspace after each build to save disk
            cleanWs()
        }
    }

}
