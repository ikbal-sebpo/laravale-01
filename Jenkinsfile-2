pipeline {
    agent any

    stages {

        stage('Deploy') {
            steps {

                withCredentials([usernamePassword(
                    credentialsId: 'almalinux-deploy',
                    usernameVariable: 'DEPLOY_USER',
                    passwordVariable: 'DEPLOY_PASSWORD'
                )]) {

                    sh '''
                        #!/bin/bash
                        set -e

                        SERVER="10.232.82.220"
                        PORT="2221"
                        REMOTE_PATH="/var/www/html/laravale-02"

                        echo "Starting deployment..."

                        echo "Copying files to server..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        scp -P "$PORT" \
                        -r "$WORKSPACE"/* \
                        "$DEPLOY_USER@$SERVER:$REMOTE_PATH/"

                        echo "Files copied successfully."

                        echo "Installing Composer dependencies..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -p "$PORT" \
                        "$DEPLOY_USER@$SERVER" \
                        "cd $REMOTE_PATH && \
                         composer install --no-dev --optimize-autoloader"

                        echo "Setting Laravel permissions..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -p "$PORT" \
                        "$DEPLOY_USER@$SERVER" \
                        "cd $REMOTE_PATH && \
                         chown -R apache:apache storage bootstrap/cache && \
                         chmod -R 775 storage bootstrap/cache"

                        echo "Clearing Laravel caches..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -p "$PORT" \
                        "$DEPLOY_USER@$SERVER" \
                        "cd $REMOTE_PATH && \
                         php artisan config:clear && \
                         php artisan cache:clear && \
                         php artisan view:clear && \
                         php artisan route:clear"

                        echo "Caching Laravel configuration..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -p "$PORT" \
                        "$DEPLOY_USER@$SERVER" \
                        "cd $REMOTE_PATH && \
                         php artisan config:cache"

                        echo "Deployment completed successfully."
                    '''
                }
            }
        }
    }
}
