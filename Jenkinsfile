pipeline {

    // ─────────────────────────────────────────────
    // AGENT — Yeh pipeline kahan chalegi?
    // Jenkins khud apne container mein chalayega
    // ─────────────────────────────────────────────
    agent any

    // ─────────────────────────────────────────────
    // ENVIRONMENT — Global variables jo poori
    // pipeline mein use honge
    // ─────────────────────────────────────────────
    environment {
        APP_NAME        = "todo-app"
        APP_VERSION     = "1.0.${BUILD_NUMBER}"  // har build ka alag version
        TESTING_SERVER  = "wenawa@35.223.36.105"
        DEPLOY_PATH     = "/home/wenawa/todo-app"
        APP_PORT        = "3000"
    }

    stages {

        // ─────────────────────────────────────────
        // STAGE 1 — CHECKOUT
        // GitHub se latest code lao
        // ─────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo "============================================"
                echo " Stage 1: GitHub se code aa raha hai..."
                echo " Version: ${APP_VERSION}"
                echo "============================================"

                // Jenkins apne aap GitHub se code pull karta hai
                // kyunki humne pipeline mein repo URL di hogi
                checkout scm

                // Confirm karo kaunsi files aayi hain
                sh 'ls -la'
                sh 'cat package.json'
            }
        }

        // ─────────────────────────────────────────
        // STAGE 2 — INSTALL DEPENDENCIES
        // npm install — sare packages download karo
        // ─────────────────────────────────────────
        stage('Install Dependencies') {
            steps {
                echo "============================================"
                echo " Stage 2: npm packages install ho rahe hain"
                echo "============================================"

                sh 'npm install'

                // Confirm karo node_modules ban gayi
                sh 'ls node_modules | head -10'
                echo "Dependencies install ho gayi ✅"
            }
        }

        // ─────────────────────────────────────────
        // STAGE 3 — TEST
        // Saare tests chalao — fail hue toh
        // pipeline ruk jayegi, deploy nahi hogi
        // ─────────────────────────────────────────
        stage('Test') {
            steps {
                echo "============================================"
                echo " Stage 3: Tests chal rahe hain..."
                echo "============================================"

                sh 'npm test'

                echo "Saare tests pass ho gaye ✅"
            }
        }

        // ─────────────────────────────────────────
        // STAGE 4 — BUILD & VERSION
        // App ka ZIP banao with version number
        // Yahi artifact hai jo hum save karenge
        // ─────────────────────────────────────────
        stage('Build & Version') {
            steps {
                echo "============================================"
                echo " Stage 4: Build ho rahi hai..."
                echo " Artifact: ${APP_NAME}-${APP_VERSION}.zip"
                echo "============================================"

                // Version file banao — app ke andar version pata chale
                sh "echo ${APP_VERSION} > version.txt"

                // ZIP banao — node_modules chod do (bohot bhari hoti hai)
                sh """
                    zip -r ${APP_NAME}-${APP_VERSION}.zip . \
                    --exclude='*.git*' \
                    --exclude='node_modules/*' \
                    --exclude='*.zip'
                """

                // Confirm karo ZIP ban gayi
                sh "ls -lh ${APP_NAME}-${APP_VERSION}.zip"
                echo "Artifact ban gaya: ${APP_NAME}-${APP_VERSION}.zip ✅"
            }
        }

        // ─────────────────────────────────────────
        // STAGE 5 — ARCHIVE ARTIFACT
        // ZIP ko Jenkins ke andar save karo
        // Jenkins dashboard pe dikhai dega
        // Rollback ke liye yahan se uthayenge
        // ─────────────────────────────────────────
        stage('Archive Artifact') {
            steps {
                echo "============================================"
                echo " Stage 5: Artifact Jenkins mein save ho raha hai"
                echo "============================================"

                // Jenkins artifact storage mein save karo
                archiveArtifacts(
                    artifacts: "${APP_NAME}-${APP_VERSION}.zip",
                    fingerprint: true,    // har artifact ka unique fingerprint
                    onlyIfSuccessful: true
                )

                echo "Artifact Jenkins mein save ho gaya ✅"
                echo "Dashboard pe dekho: Build #${BUILD_NUMBER} > Artifacts"
            }
        }

        // ─────────────────────────────────────────
        // STAGE 6 — DEPLOY TO TESTING SERVER
        // ZIP testing server pe bhejo
        // Wahan unzip karo aur app start karo
        // PM2 se app background mein chalegi
        // ─────────────────────────────────────────
        stage('Deploy to Testing Server') {
            steps {
                echo "============================================"
                echo " Stage 6: Testing server pe deploy ho raha hai"
                echo " Server: ${TESTING_SERVER}"
                echo " Path: ${DEPLOY_PATH}"
                echo "============================================"

                // Step 1: Testing server pe folder banao
                sh """
                    ssh -o StrictHostKeyChecking=no ${TESTING_SERVER} '
                        mkdir -p ${DEPLOY_PATH}
                        mkdir -p ${DEPLOY_PATH}/releases
                    '
                """

                // Step 2: ZIP testing server pe bhejo (SCP)
                sh """
                    scp -o StrictHostKeyChecking=no \
                        ${APP_NAME}-${APP_VERSION}.zip \
                        ${TESTING_SERVER}:${DEPLOY_PATH}/releases/
                """

                // Step 3: Testing server pe unzip karo aur app start karo
                sh """
                    ssh -o StrictHostKeyChecking=no ${TESTING_SERVER} '

                        # Release folder mein jao
                        cd ${DEPLOY_PATH}/releases

                        # ZIP unzip karo
                        unzip -o ${APP_NAME}-${APP_VERSION}.zip \
                              -d ${APP_NAME}-${APP_VERSION}

                        # Latest folder update karo (symlink)
                        ln -sfn \
                            ${DEPLOY_PATH}/releases/${APP_NAME}-${APP_VERSION} \
                            ${DEPLOY_PATH}/current

                        # Current folder mein jao
                        cd ${DEPLOY_PATH}/current

                        # Dependencies install karo
                        npm install --production

                        # PM2 se app restart/start karo
                        pm2 delete todo-app 2>/dev/null || true
                        APP_VERSION=${APP_VERSION} pm2 start src/app.js \
                            --name "todo-app" \
                            --env production

                        # PM2 status dekho
                        pm2 list
                    '
                """

                echo "Deploy ho gaya ✅"
            }
        }

        // ─────────────────────────────────────────
        // STAGE 7 — VERIFY DEPLOYMENT
        // Deploy hone ke baad check karo
        // kya app sach mein chal rahi hai?
        // Curl se health check karenge
        // ─────────────────────────────────────────
        stage('Verify Deployment') {
            steps {
                echo "============================================"
                echo " Stage 7: Deployment verify ho rahi hai..."
                echo "============================================"

                // 5 second wait karo — app start hone do
                sh 'sleep 5'

                // Health check — agar fail hua toh pipeline fail
                sh """
                    curl -f http://35.223.36.105:${APP_PORT}/health || \
                    (echo "Health check FAIL ❌" && exit 1)
                """

                // Version bhi check karo
                sh """
                    echo "Response:"
                    curl -s http://35.223.36.105:${APP_PORT}/health
                """

                echo "App chal rahi hai ✅"
                echo "URL: http://35.223.36.105:${APP_PORT}"
            }
        }

        // ─────────────────────────────────────────
        // STAGE 8 — SAVE TO REMOTE DISK
        // Artifact ko testing server ki disk pe
        // alag folder mein bhi save karo
        // Yeh long-term backup hai
        // ─────────────────────────────────────────
        stage('Save to Remote Disk') {
            steps {
                echo "============================================"
                echo " Stage 8: Remote disk pe save ho raha hai"
                echo "============================================"

                sh """
                    ssh -o StrictHostKeyChecking=no ${TESTING_SERVER} '

                        # Backup folder banao
                        mkdir -p /home/wenawa/artifacts

                        # Current release ka ZIP backup mein copy karo
                        cp ${DEPLOY_PATH}/releases/${APP_NAME}-${APP_VERSION}.zip \
                           /home/wenawa/artifacts/

                        # Sari saved artifacts dekho
                        echo "Saved artifacts:"
                        ls -lh /home/wenawa/artifacts/
                    '
                """

                echo "Disk pe save ho gaya ✅"
                echo "Location: wenawa@35.223.36.105:/home/wenawa/artifacts/"
            }
        }

    } // end stages

    // ─────────────────────────────────────────────
    // POST — Pipeline khatam hone ke baad
    // Pass ho ya fail — yahan actions hote hain
    // ─────────────────────────────────────────────
    post {

        success {
            echo "================================================"
            echo " PIPELINE SUCCESS ✅"
            echo " App: ${APP_NAME}"
            echo " Version: ${APP_VERSION}"
            echo " Live URL: http://35.223.36.105:${APP_PORT}"
            echo " Artifacts: Dashboard > Build #${BUILD_NUMBER}"
            echo "================================================"
        }

        failure {
            echo "================================================"
            echo " PIPELINE FAILED ❌"
            echo " Build #${BUILD_NUMBER} fail ho gayi"
            echo " Logs dekho upar — konsa stage fail hua"
            echo "================================================"
        }

        always {
            // Workspace clean karo — disk space bachao
            echo "Workspace clean ho raha hai..."
            cleanWs() 
            app.get('/health', (req, res) => {
    res.status(200).json({ 
        status: 'ok',
        version: '1.0.1'
    })
})
        }
    }

} // end pipeline
