
pipeline {
    agent any

    tools {
        nodejs 'Node-js-26'
    }

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }

        stage('Tests') {
            steps {
                sh 'npm run test:ci'
            }
            post {
                always {
                    junit testResults: 'reports/junit/junit.xml',
                          allowEmptyResults: true
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Package') {
            steps {
                sh """
                    tar -czf node-demo-${BUILD_NUMBER}.tar.gz \
                        dist package.json package-lock.json
                """
                archiveArtifacts artifacts: "node-demo-${BUILD_NUMBER}.tar.gz"
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    set -eu

                    DEPLOY_ROOT=/opt/cicd-demo/node
                    RELEASE_DIR="$DEPLOY_ROOT/releases/$BUILD_NUMBER"
                    ARTIFACT="node-demo-${BUILD_NUMBER}.tar.gz"

                    mkdir -p "$RELEASE_DIR"
                    tar -xzf "$ARTIFACT" -C "$RELEASE_DIR"

                    cd "$RELEASE_DIR"
                    npm ci --omit=dev

                    if [ -f "$DEPLOY_ROOT/app.pid" ]; then
                        OLD_PID=$(cat "$DEPLOY_ROOT/app.pid")
                        if kill -0 "$OLD_PID" 2>/dev/null; then
                            kill "$OLD_PID"
                            sleep 2
                        fi
                    fi

                    ln -sfn "$RELEASE_DIR" "$DEPLOY_ROOT/current"

                    JENKINS_NODE_COOKIE=dontKillMe \
                    APP_VERSION="$BUILD_NUMBER" \
                    PORT=3000 \
                    nohup node "$DEPLOY_ROOT/current/dist/server.js" \
                        > "$DEPLOY_ROOT/app.log" 2>&1 &

                    echo $! > "$DEPLOY_ROOT/app.pid"
                '''
            }
        }

        stage('Smoke Test') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    sleep 3
                    curl --fail --show-error \
                        http://127.0.0.1:3000/health
                '''
            }
        }
    }

    post {
        success {
            echo 'PIPELINE SUCCESSFUL'
        }
        failure {
            echo 'PIPELINE FAILED - check the failed stage'
        }
    }
}
