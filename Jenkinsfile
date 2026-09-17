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

                        SERVER="10.232.82.220"
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


stage('Deploy Production') {
    steps {

        input message: 'Staging deployment successful. Deploy to Production?',
              ok: 'Deploy Production'

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
                echo "Starting PRODUCTION deployment..."
                echo "Server: $SERVER"
                echo "===================================="

                echo "Testing SSH connection..."

                sshpass -p "$DEPLOY_PASSWORD" \
                ssh -o StrictHostKeyChecking=no \
                    -p "$PORT" \
                    "$DEPLOY_USER@$SERVER" \
                    "echo SSH connection successful"

                echo "Creating remote directory..."

                sshpass -p "$DEPLOY_PASSWORD" \
                ssh -o StrictHostKeyChecking=no \
                    -p "$PORT" \
                    "$DEPLOY_USER@$SERVER" \
                    "mkdir -p $REMOTE_PATH"

                echo "Copying Laravel application..."

                sshpass -p "$DEPLOY_PASSWORD" \
                scp -o StrictHostKeyChecking=no \
                    -P "$PORT" \
                    -r \
                    "$WORKSPACE/app" \
                    "$WORKSPACE/bootstrap" \
                    "$WORKSPACE/config" \
                    "$WORKSPACE/database" \
                    "$WORKSPACE/public" \
                    "$WORKSPACE/resources" \
                    "$WORKSPACE/routes" \
                    "$WORKSPACE/storage" \
                    "$WORKSPACE/tests" \
                    "$WORKSPACE/artisan" \
                    "$WORKSPACE/composer.json" \
                    "$WORKSPACE/composer.lock" \
                    "$DEPLOY_USER@$SERVER:$REMOTE_PATH/"

                echo "Files copied successfully."

                echo "Installing Composer dependencies..."

                sshpass -p "$DEPLOY_PASSWORD" \
                ssh -o StrictHostKeyChecking=no \
                    -p "$PORT" \
                    "$DEPLOY_USER@$SERVER" \
                    "cd $REMOTE_PATH && \
                     composer install --no-dev --optimize-autoloader"

                echo "Setting Laravel permissions..."

                sshpass -p "$DEPLOY_PASSWORD" \
                ssh -o StrictHostKeyChecking=no \
                    -p "$PORT" \
                    "$DEPLOY_USER@$SERVER" \
                    "cd $REMOTE_PATH && \
                     chown -R apache:apache storage bootstrap/cache && \
                     chmod -R 775 storage bootstrap/cache"

                echo "Clearing Laravel caches..."

                sshpass -p "$DEPLOY_PASSWORD" \
                ssh -o StrictHostKeyChecking=no \
                    -p "$PORT" \
                    "$DEPLOY_USER@$SERVER" \
                    "cd $REMOTE_PATH && \
                     php artisan config:clear && \
                     php artisan cache:clear && \
                     php artisan view:clear && \
                     php artisan route:clear"

                echo "Caching Laravel configuration..."

                sshpass -p "$DEPLOY_PASSWORD" \
                ssh -o StrictHostKeyChecking=no \
                    -p "$PORT" \
                    "$DEPLOY_USER@$SERVER" \
                    "cd $REMOTE_PATH && \
                     php artisan config:cache"

                echo "===================================="
                echo "PRODUCTION deployment completed!"
                echo "===================================="
            '''
        }
    }
}
