pipeline {

    agent any

    parameters {

        string(
            name: 'PRODUCTION_VERSION',
            defaultValue: '',
            description: 'Git tag/version to deploy to Production. Example: v1.0.0'
        )

        booleanParam(
            name: 'DEPLOY_PRODUCTION',
            defaultValue: false,
            description: 'Enable Production deployment'
        )
    }

    environment {

        /*
         * ============================================================
         * STAGING
         * ============================================================
         */
        STAGING_SERVER = '10.232.82.220'
        STAGING_PORT = '2221'
        STAGING_PATH = '/var/www/html/laravale-02'


        /*
         * ============================================================
         * PRODUCTION
         * ============================================================
         */
        PRODUCTION_SERVER = '10.232.82.222'
        PRODUCTION_PORT = '2221'
        PRODUCTION_PATH = '/var/www/html/laravale-02'


        /*
         * ============================================================
         * ARTIFACT
         * ============================================================
         */
        ARTIFACT_DIR = "${WORKSPACE}/artifacts"
    }


    stages {

        // ============================================================
        // CHECKOUT
        // ============================================================
        stage('Checkout') {

            steps {

                checkout scm

                sh '''
                    set -e

                    echo "===================================="
                    echo "Git Information"
                    echo "===================================="

                    git log -1 --oneline

                    echo ""
                    echo "Commit:"
                    git rev-parse HEAD

                    echo ""
                    echo "Tag:"
                    git describe --tags --always
                '''
            }
        }


        // ============================================================
        // BUILD
        // ============================================================
        stage('Build') {

            steps {

                sh '''
                    set -e

                    echo "===================================="
                    echo "BUILD"
                    echo "===================================="

                    php -v
                    composer --version

                    echo ""
                    echo "Installing Composer dependencies..."

                    composer install \
                        --prefer-dist \
                        --optimize-autoloader \
                        --no-interaction

                    echo ""
                    echo "Build completed successfully."
                '''
            }
        }


        // ============================================================
        // TEST
        // ============================================================
        stage('Test') {

            steps {

                sh '''
                    set -e

                    echo "===================================="
                    echo "TEST"
                    echo "===================================="

                    echo "Preparing Laravel test environment..."

                    cp .env.example .env

                    php artisan key:generate --force

                    echo ""
                    echo "Running Laravel tests..."

                    php artisan test

                    echo ""
                    echo "Tests passed successfully."
                '''
            }
        }


        // ============================================================
        // CREATE ARTIFACT
        // ============================================================
        stage('Create Artifact') {

            steps {

                sh '''
                    set -e

                    echo "===================================="
                    echo "CREATE ARTIFACT"
                    echo "===================================="

                    rm -rf "$ARTIFACT_DIR"

                    mkdir -p "$ARTIFACT_DIR"


                    # ------------------------------------------------
                    # Get Git version
                    # ------------------------------------------------

                    VERSION=$(git describe --tags --exact-match 2>/dev/null || true)

                    if [ -z "$VERSION" ]; then
                        VERSION="commit-$(git rev-parse --short HEAD)"
                    fi

                    echo "Artifact Version: $VERSION"


                    # ------------------------------------------------
                    # Save version information
                    # ------------------------------------------------

                    echo "$VERSION" > "$ARTIFACT_DIR/VERSION"

                    git rev-parse HEAD > "$ARTIFACT_DIR/COMMIT"


                    # ------------------------------------------------
                    # Create application archive
                    # ------------------------------------------------

                    tar \
                        --exclude='.git' \
                        --exclude='artifacts' \
                        --exclude='.env' \
                        -czf "$ARTIFACT_DIR/app-${VERSION}.tar.gz" .


                    echo ""
                    echo "Artifact created:"
                    ls -lh "$ARTIFACT_DIR"


                    echo ""
                    echo "===================================="
                    echo "Publishing Jenkins artifact..."
                    echo "===================================="
                '''


                archiveArtifacts(
                    artifacts: 'artifacts/*',
                    fingerprint: true
                )
            }
        }


        // ============================================================
        // DEPLOY STAGING
        // Automatically deploy same artifact
        // ============================================================
        stage('Deploy Staging') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'almalinux-deploy',
                        usernameVariable: 'DEPLOY_USER',
                        passwordVariable: 'DEPLOY_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "===================================="
                        echo "STAGING DEPLOYMENT"
                        echo "===================================="


                        VERSION=$(cat "$ARTIFACT_DIR/VERSION")

                        ARTIFACT="$ARTIFACT_DIR/app-${VERSION}.tar.gz"


                        echo "Version : $VERSION"
                        echo "Server  : $STAGING_SERVER"
                        echo "Path    : $STAGING_PATH"


                        # ------------------------------------------------
                        # Create temporary deployment directory
                        # ------------------------------------------------

                        TEMP_DIR="/tmp/laravale-${BUILD_NUMBER}"

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh \
                            -p "$STAGING_PORT" \
                            "$DEPLOY_USER@$STAGING_SERVER" \
                            "rm -rf '$TEMP_DIR' && mkdir -p '$TEMP_DIR'"


                        # ------------------------------------------------
                        # Upload SAME artifact
                        # ------------------------------------------------

                        sshpass -p "$DEPLOY_PASSWORD" \
                        scp \
                            -P "$STAGING_PORT" \
                            "$ARTIFACT" \
                            "$DEPLOY_USER@$STAGING_SERVER:$TEMP_DIR/app.tar.gz"


                        # ------------------------------------------------
                        # Extract artifact
                        # ------------------------------------------------

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh \
                            -p "$STAGING_PORT" \
                            "$DEPLOY_USER@$STAGING_SERVER" \
                            "cd '$TEMP_DIR' && \
                             tar -xzf app.tar.gz"


                        # ------------------------------------------------
                        # Deploy application
                        # ------------------------------------------------

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh \
                            -p "$STAGING_PORT" \
                            "$DEPLOY_USER@$STAGING_SERVER" \
                            "rm -rf '$STAGING_PATH'/* && \
                             cp -a '$TEMP_DIR'/.[!.]* '$STAGING_PATH'/ 2>/dev/null || true && \
                             cp -a '$TEMP_DIR'/* '$STAGING_PATH'/ 2>/dev/null || true"


                        # ------------------------------------------------
                        # Laravel permissions
                        # ------------------------------------------------

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh \
                            -p "$STAGING_PORT" \
                            "$DEPLOY_USER@$STAGING_SERVER" \
                            "cd '$STAGING_PATH' && \
                             chown -R apache:apache storage bootstrap/cache && \
                             chmod -R 775 storage bootstrap/cache"


                        # ------------------------------------------------
                        # Laravel cache
                        # ------------------------------------------------

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh \
                            -p "$STAGING_PORT" \
                            "$DEPLOY_USER@$STAGING_SERVER" \
                            "cd '$STAGING_PATH' && \
                             php artisan optimize:clear && \
                             php artisan config:cache"


                        # ------------------------------------------------
                        # Save deployed version
                        # ------------------------------------------------

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh \
                            -p "$STAGING_PORT" \
                            "$DEPLOY_USER@$STAGING_SERVER" \
                            "echo '$VERSION' > '$STAGING_PATH/VERSION'"


                        # ------------------------------------------------
                        # Cleanup
                        # ------------------------------------------------

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh \
                            -p "$STAGING_PORT" \
                            "$DEPLOY_USER@$STAGING_SERVER" \
                            "rm -rf '$TEMP_DIR'"


                        echo ""
                        echo "===================================="
                        echo "STAGING DEPLOYMENT SUCCESSFUL"
                        echo "Version: $VERSION"
                        echo "===================================="
                    '''
                }
            }
        }


        // ============================================================
        // PRODUCTION DEPLOY
        // Manual
        // ============================================================
        stage('Deploy Production') {

            when {

                expression {
                    return params.DEPLOY_PRODUCTION &&
                           params.PRODUCTION_VERSION?.trim()
                }
            }


            steps {

                script {

                    echo "===================================="
                    echo "PRODUCTION DEPLOYMENT"
                    echo "===================================="

                    echo "Selected Version: ${params.PRODUCTION_VERSION}"


                    // ------------------------------------------------
                    // Confirm deployment
                    // ------------------------------------------------

                    input(
                        message: "Deploy ${params.PRODUCTION_VERSION} to PRODUCTION?",
                        ok: "Deploy Production"
                    )


                    // ------------------------------------------------
                    // Find artifact
                    // ------------------------------------------------

                    def version = params.PRODUCTION_VERSION.trim()

                    def artifactPath =
                        "${env.JENKINS_HOME}/jobs/${env.JOB_NAME}/builds"

                    echo "Looking for artifact version: ${version}"


                    withCredentials([
                        usernamePassword(
                            credentialsId: 'almalinux-deploy',
                            usernameVariable: 'DEPLOY_USER',
                            passwordVariable: 'DEPLOY_PASSWORD'
                        )
                    ]) {

                        sh """
                            set -e

                            echo "===================================="
                            echo "PRODUCTION DEPLOYMENT"
                            echo "===================================="

                            VERSION="${version}"

                            echo "Selected Version: \$VERSION"


                            echo ""
                            echo "Searching Jenkins archived artifacts..."


                            # ------------------------------------------------
                            # Find artifact from previous successful build
                            # ------------------------------------------------

                            ARTIFACT_FILE=""

                            for BUILD_DIR in \$(find "\$JENKINS_HOME/jobs" -type f -name "app-\${VERSION}.tar.gz" 2>/dev/null | sort -r); do
                                ARTIFACT_FILE="\$BUILD_DIR"
                                break
                            done


                            if [ -z "\$ARTIFACT_FILE" ]; then

                                echo ""
                                echo "ERROR: Artifact not found for version \$VERSION"

                                exit 1
                            fi


                            echo ""
                            echo "Artifact found:"
                            echo "\$ARTIFACT_FILE"


                            # ------------------------------------------------
                            # Temporary directory
                            # ------------------------------------------------

                            TEMP_DIR="/tmp/laravale-production-${BUILD_NUMBER}"


                            sshpass -p "\$DEPLOY_PASSWORD" \\
                            ssh \\
                                -p "\$PRODUCTION_PORT" \\
                                "\$DEPLOY_USER@\$PRODUCTION_SERVER" \\
                                "rm -rf '\$TEMP_DIR' && mkdir -p '\$TEMP_DIR'"


                            # ------------------------------------------------
                            # Upload SAME artifact
                            # ------------------------------------------------

                            sshpass -p "\$DEPLOY_PASSWORD" \\
                            scp \\
                                -P "\$PRODUCTION_PORT" \\
                                "\$ARTIFACT_FILE" \\
                                "\$DEPLOY_USER@\$PRODUCTION_SERVER:\$TEMP_DIR/app.tar.gz"


                            # ------------------------------------------------
                            # Extract
                            # ------------------------------------------------

                            sshpass -p "\$DEPLOY_PASSWORD" \\
                            ssh \\
                                -p "\$PRODUCTION_PORT" \\
                                "\$DEPLOY_USER@\$PRODUCTION_SERVER" \\
                                "cd '\$TEMP_DIR' && tar -xzf app.tar.gz"


                            # ------------------------------------------------
                            # Deploy
                            # ------------------------------------------------

                            sshpass -p "\$DEPLOY_PASSWORD" \\
                            ssh \\
                                -p "\$PRODUCTION_PORT" \\
                                "\$DEPLOY_USER@\$PRODUCTION_SERVER" \\
                                "rm -rf '$PRODUCTION_PATH'/* && \\
                                 cp -a '\$TEMP_DIR'/.[!.]* '$PRODUCTION_PATH'/ 2>/dev/null || true && \\
                                 cp -a '\$TEMP_DIR'/* '$PRODUCTION_PATH'/ 2>/dev/null || true"


                            # ------------------------------------------------
                            # Permissions
                            # ------------------------------------------------

                            sshpass -p "\$DEPLOY_PASSWORD" \\
                            ssh \\
                                -p "\$PRODUCTION_PORT" \\
                                "\$DEPLOY_USER@\$PRODUCTION_SERVER" \\
                                "cd '$PRODUCTION_PATH' && \\
                                 chown -R apache:apache storage bootstrap/cache && \\
                                 chmod -R 775 storage bootstrap/cache"


                            # ------------------------------------------------
                            # Laravel cache
                            # ------------------------------------------------

                            sshpass -p "\$DEPLOY_PASSWORD" \\
                            ssh \\
                                -p "\$PRODUCTION_PORT" \\
                                "\$DEPLOY_USER@\$PRODUCTION_SERVER" \\
                                "cd '$PRODUCTION_PATH' && \\
                                 php artisan optimize:clear && \\
                                 php artisan config:cache"


                            # ------------------------------------------------
                            # Save deployed version
                            # ------------------------------------------------

                            sshpass -p "\$DEPLOY_PASSWORD" \\
                            ssh \\
                                -p "\$PRODUCTION_PORT" \\
                                "\$DEPLOY_USER@\$PRODUCTION_SERVER" \\
                                "echo '\$VERSION' > '$PRODUCTION_PATH/VERSION'"


                            # ------------------------------------------------
                            # Cleanup
                            # ------------------------------------------------

                            sshpass -p "\$DEPLOY_PASSWORD" \\
                            ssh \\
                                -p "\$PRODUCTION_PORT" \\
                                "\$DEPLOY_USER@\$PRODUCTION_SERVER" \\
                                "rm -rf '\$TEMP_DIR'"


                            echo ""
                            echo "===================================="
                            echo "PRODUCTION DEPLOYMENT SUCCESSFUL"
                            echo "Version: \$VERSION"
                            echo "===================================="
                        """
                    }
                }
            }
        }
    }


    // ================================================================
    // POST ACTIONS
    // ================================================================
// ================================================================
// POST ACTIONS
// ================================================================
post {

    success {

        echo """
        ====================================
        PIPELINE SUCCESS
        ====================================
        Build: ${env.BUILD_NUMBER}
        Job:   ${env.JOB_NAME}
        ====================================
        """

        emailext(
            to: 'i.hossain@sebpo.com',
            subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: """
Hello,

The Jenkins pipeline completed successfully.

====================================
PIPELINE SUCCESS
====================================

Job       : ${env.JOB_NAME}
Build     : #${env.BUILD_NUMBER}
Status    : SUCCESS
Version   : ${params.PRODUCTION_VERSION ?: 'N/A'}

Build URL :
${env.BUILD_URL}

====================================

Regards,
Jenkins
"""
        )
    }


    failure {

        echo """
        ====================================
        PIPELINE FAILED
        ====================================
        Build: ${env.BUILD_NUMBER}
        Job:   ${env.JOB_NAME}
        ====================================
        """

        emailext(
            to: 'i.hossain@sebpo.com',
            subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: """
Hello,

The Jenkins pipeline has failed.

====================================
PIPELINE FAILED
====================================

Job       : ${env.JOB_NAME}
Build     : #${env.BUILD_NUMBER}
Status    : FAILED
Version   : ${params.PRODUCTION_VERSION ?: 'N/A'}

Build URL :
${env.BUILD_URL}

Please check the Jenkins console output for details.

====================================

Regards,
Jenkins
"""
        )
    }
}
}

