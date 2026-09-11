pipeline {
    agent any

    environment {
        REMOTE_HOST = '10.232.82.214'
        REMOTE_USER = 'ubuntu'
        APP_DIR     = '/var/www/html/my-project'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Deploy Application') {
            steps {
                sshagent(['6804e923-6c7a-4ce7-a15f-e5d2464ef74e']) {
                    sh """
                        rsync -avz --delete \
                        --exclude='.git' \
                        --exclude='.env' \
                        --exclude='storage/logs/*' \
                        ./ ${REMOTE_USER}@${REMOTE_HOST}:${APP_DIR}/
                    """
                }
            }
        }

        stage('Install Composer Dependencies') {
            steps {
                sshagent(['6804e923-6c7a-4ce7-a15f-e5d2464ef74e']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no \
                        ${REMOTE_USER}@${REMOTE_HOST} '
                            cd ${APP_DIR} &&
                            composer install \
                            --no-dev \
                            --optimize-autoloader
                        '
                    """
                }
            }
        }

        stage('Laravel Optimization') {
            steps {
                sshagent(['6804e923-6c7a-4ce7-a15f-e5d2464ef74e']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no \
                        ${REMOTE_USER}@${REMOTE_HOST} '
                            cd ${APP_DIR} &&
                            php artisan storage:link &&
                            php artisan config:clear &&
                            php artisan cache:clear &&
                            php artisan route:clear &&
                            php artisan view:clear &&
                            php artisan config:cache &&
                            php artisan route:cache &&
                            php artisan view:cache
                        '
                    """
                }
            }
        }

        stage('Database Migration') {
            steps {
                sshagent(['6804e923-6c7a-4ce7-a15f-e5d2464ef74e']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no \
                        ${REMOTE_USER}@${REMOTE_HOST} '
                            cd ${APP_DIR} &&
                            php artisan migrate --force
                        '
                    """
                }
            }
        }

        stage('Set Permissions') {
            steps {
                sshagent(['6804e923-6c7a-4ce7-a15f-e5d2464ef74e']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no \
                        ${REMOTE_USER}@${REMOTE_HOST} '
                            cd ${APP_DIR} &&
                            chown -R www-data:www-data storage bootstrap/cache &&
                            chmod -R 775 storage bootstrap/cache
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Laravel Application সফলভাবে Deploy হয়েছে।'
        }

        failure {
            echo 'Laravel Deployment ব্যর্থ হয়েছে। Jenkins Console Log চেক করুন।'
        }
    }
}
