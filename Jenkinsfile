pipeline {

    agent any

    environment {

        STAGING_SERVER = '10.232.82.220'
        PROD_SERVER    = '10.232.82.222'

        SSH_PORT       = '2221'
        APP_DIR        = '/var/www/html/laravale-02'
        PACKAGE        = 'build.tar.gz'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {

                sh '''
                    set -e

                    echo "PHP Version:"
                    php -v

                    echo "Composer Version:"
                    composer --version

                    echo "Installing Composer dependencies..."

                    composer install \
                        --no-dev \
                        --prefer-dist \
                        --optimize-autoloader

                    echo "Running Laravel tests..."

                    php artisan test
                '''
            }
        }

        stage('Package') {
            steps {

                sh '''
                    set -e

                    echo "Creating deployment package..."

                    tar --exclude='.git' \
                        --exclude='.env' \
                        --exclude='build.tar.gz' \
                        --exclude='node_modules' \
                        -czf build.tar.gz .

                    ls -lh build.tar.gz
                '''
            }
        }

        stage('Deploy Staging') {
            steps {

                withCredentials([usernamePassword(
                    credentialsId: 'almalinux-deploy',
                    usernameVariable: 'DEPLOY_USER',
                    passwordVariable: 'DEPLOY_PASSWORD'
                )]) {

                    sh '''
                        set -e

                        echo "===================================="
                        echo "Deploying to STAGING"
                        echo "Server: $STAGING_SERVER"
                        echo "===================================="

                        sshpass -p "$DEPLOY_PASSWORD" \
                        scp -o StrictHostKeyChecking=no \
                            -P "$SSH_PORT" \
                            "$PACKAGE" \
                            "$DEPLOY_USER@$STAGING_SERVER:/tmp/build.tar.gz"

                        echo "Package uploaded to Staging."
                    '''
                }
            }
        }

        stage('Extract & Configure Staging') {
            steps {

                withCredentials([usernamePassword(
                    credentialsId: 'almalinux-deploy',
                    usernameVariable: 'DEPLOY_USER',
                    passwordVariable: 'DEPLOY_PASSWORD'
                )]) {

                    sh '''
                        set -e

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -o StrictHostKeyChecking=no \
                            -p "$SSH_PORT" \
                            "$DEPLOY_USER@$STAGING_SERVER" \
                            "cd $APP_DIR && \
                             tar -xzf /tmp/build.tar.gz && \
                             composer install --no-dev --optimize-autoloader && \
                             chown -R apache:apache storage bootstrap/cache && \
                             chmod -R 775 storage bootstrap/cache && \
                             php artisan config:clear && \
                             php artisan cache:clear && \
                             php artisan view:clear && \
                             php artisan route:clear && \
                             php artisan config:cache"

                        echo "Staging deployment completed."
                    '''
                }
            }
        }

        stage('Staging Health Check') {
            steps {

                sh '''
                    set -e

                    echo "Checking Staging server..."

                    curl -f -I --max-time 10 \
                        http://10.232.82.220

                    echo "Staging health check passed."
                '''
            }
        }

        stage('SysAdmin Approval') {
            steps {

                input(
                    message: 'Staging Server OK? Deploy to Production?',
                    ok: 'YES - DEPLOY PRODUCTION'
                )
            }
        }

        stage('Deploy Production') {
            steps {

                withCredentials([usernamePassword(
                    credentialsId: 'almalinux-deploy',
                    usernameVariable: 'DEPLOY_USER',
                    passwordVariable: 'DEPLOY_PASSWORD'
                )]) {

                    sh '''
                        set -e

                        echo "===================================="
                        echo "Deploying to PRODUCTION"
                        echo "Server: $PROD_SERVER"
                        echo "===================================="

                        echo "Testing Production SSH..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -o StrictHostKeyChecking=no \
                            -p "$SSH_PORT" \
                            "$DEPLOY_USER@$PROD_SERVER" \
                            "echo Production SSH connection successful"

                        echo "Uploading package..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        scp -o StrictHostKeyChecking=no \
                            -P "$SSH_PORT" \
                            "$PACKAGE" \
                            "$DEPLOY_USER@$PROD_SERVER:/tmp/build.tar.gz"

                        echo "Package uploaded to Production."

                        echo "Extracting application..."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -o StrictHostKeyChecking=no \
                            -p "$SSH_PORT" \
                            "$DEPLOY_USER@$PROD_SERVER" \
                            "cd $APP_DIR && \
                             tar -xzf /tmp/build.tar.gz && \
                             composer install --no-dev --optimize-autoloader && \
                             chown -R apache:apache storage bootstrap/cache && \
                             chmod -R 775 storage bootstrap/cache && \
                             php artisan config:clear && \
                             php artisan cache:clear && \
                             php artisan view:clear && \
                             php artisan route:clear && \
                             php artisan config:cache"

                        echo "Production deployment completed successfully."
                    '''
                }
            }
        }

        stage('Production Health Check') {
            steps {

                sh '''
                    set -e

                    echo "Checking Production server..."

                    curl -f -I --max-time 10 \
                        http://10.232.82.222

                    echo "Production health check passed."
                '''
            }
        }
    }
}
