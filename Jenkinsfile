pipeline {
    agent any

    stages {

        stage('Build & Test') {
            steps {
                sh '''
                    set -e

                    echo "===================================="
                    echo "PHP Version"
                    echo "===================================="
                    php -v

                    echo "===================================="
                    echo "Composer Version"
                    echo "===================================="
                    composer --version

                    echo "===================================="
                    echo "Installing Composer dependencies"
                    echo "===================================="
                    composer install --prefer-dist --optimize-autoloader

                    echo "===================================="
                    echo "Preparing Laravel Test Environment"
                    echo "===================================="

                    cp .env.example .env
                    php artisan key:generate --force

                    echo "===================================="
                    echo "Running Laravel Tests"
                    echo "===================================="

                    php artisan test

                    echo "===================================="
                    echo "Build & Test completed successfully"
                    echo "===================================="
                '''
            }
        }

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

                        SERVER="10.232.82.222"
                        PORT="2221"
                        REMOTE_PATH="/var/www/html/laravale-02"

                        echo "===================================="
                        echo "Starting STAGING deployment..."
                        echo "Server: $SERVER"
                        echo "Port: $PORT"
                        echo "Remote Path: $REMOTE_PATH"
                        echo "===================================="

                        echo "Copying files to staging server..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        scp -P "$PORT" \
                        -r \
                        "$WORKSPACE"/* \
                        "$DEPLOY_USER@$SERVER:$REMOTE_PATH/"

                        echo "Files copied successfully."

                        echo "Installing production dependencies..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -p "$PORT" \
                        "$DEPLOY_USER@$SERVER" \
                        "cd '$REMOTE_PATH' && \
                         composer install --no-dev --prefer-dist --optimize-autoloader"

                        echo "Setting Laravel permissions..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -p "$PORT" \
                        "$DEPLOY_USER@$SERVER" \
                        "cd '$REMOTE_PATH' && \
                         chown -R apache:apache storage bootstrap/cache && \
                         chmod -R 775 storage bootstrap/cache"

                        echo "Clearing Laravel caches..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -p "$PORT" \
                        "$DEPLOY_USER@$SERVER" \
                        "cd '$REMOTE_PATH' && \
                         php artisan config:clear && \
                         php artisan cache:clear && \
                         php artisan view:clear && \
                         php artisan route:clear"

                        echo "Caching Laravel configuration..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -p "$PORT" \
                        "$DEPLOY_USER@$SERVER" \
                        "cd '$REMOTE_PATH' && \
                         php artisan config:cache"

                        echo "===================================="
                        echo "STAGING deployment completed successfully."
                        echo "===================================="
                    '''
                }
            }
        }
    }
}
