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

                        sshpass -p "$DEPLOY_PASSWORD" \
                        scp -P "$PORT" \
                        -r "$WORKSPACE"/* \
                        "$DEPLOY_USER@$SERVER:$REMOTE_PATH/"

                        echo "Files copied successfully."

                        sshpass -p "$DEPLOY_PASSWORD" \
                        ssh -p "$PORT" \
                        "$DEPLOY_USER@$SERVER" \
                        "cd $REMOTE_PATH && \
                         php artisan optimize:clear && \
                         php artisan config:cache"

                        echo "Deployment completed successfully."
                    '''
                }
            }
        }
    }
}
