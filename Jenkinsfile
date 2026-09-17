pipeline {
    agent any

stages {

        stage('Build') {
            steps {
                sh 'echo "Hello World"'

                sh '''
                    echo "Multiline shell steps works too"
                    echo "Workspace:"
                    ls -lah
                '''
            }
        }


    stages {

        stage('Deploy Staging') {
            steps {

                withCredentials([usernamePassword(
                    credentialsId: 'almalinux-deploy',
                    usernameVariable: 'DEPLOY_USER',
                    passwordVariable: 'DEPLOY_PASSWORD'
                )]) {

                    sh '''
                        #!/bin/bash
                        set -e

                        SERVER="10.232.82.222"
                        PORT="2221"
                        REMOTE_PATH="/var/www/html/laravale-02"

                        echo "===================================="
                        echo "Starting STAGING deployment..."
                        echo "Server: $SERVER"
                        echo "===================================="

                        sshpass -p "$DEPLOY_PASSWORD" \
                        scp -P "$PORT" \
                        -r "$WORKSPACE"/* \
                        "$DEPLOY_USER@$SERVER:$REMOTE_PATH/"

                        echo "Files copied to STAGING."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -p "$PORT" \
                        "$DEPLOY_USER@$SERVER" \
                        "cd $REMOTE_PATH && \
                         composer install --no-dev --optimize-autoloader && \
                         chown -R apache:apache storage bootstrap/cache && \
                         chmod -R 775 storage bootstrap/cache && \
                         php artisan config:clear && \
                         php artisan cache:clear && \
                         php artisan view:clear && \
                         php artisan route:clear && \
                         php artisan config:cache"

                        echo "STAGING deployment completed successfully."
                    '''
                }
            }
        }

