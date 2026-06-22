pipeline {
    agent any

    environment {
        CONTAINER_NAME  = 'minio-filesystem'
        MINIO_API_PORT  = '9000'
        MINIO_UI_PORT   = '9001'
        MINIO_BUCKET    = 'biopatternsg-kb'
        NETWORK         = 'general-network'
    }

    stages {

        stage('Create Network') {
            steps {
                script {
                    echo "--- Ensuring Docker network '${NETWORK}' exists ---"
                    sh "docker network create ${NETWORK} || true"
                }
            }
        }

        stage('Deploy MinIO') {
            steps {
                script {
                    echo "--- Deploying MinIO container ---"
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'minio-credentials', 
                            usernameVariable: 'MINIO_USER',
                            passwordVariable: 'MINIO_PASSWORD'
                        )
                    ]) {
                        sh 'docker compose up -d'
                    }
                    echo "--- MinIO deployed successfully ---"
                }
            }
        }

        stage('Wait for MinIO Ready') {
            steps {
                script {
                    echo "--- Waiting for MinIO to be ready ---"
                    sh """
                        for i in \$(seq 1 15); do
                            if docker exec ${CONTAINER_NAME} mc ready local 2>/dev/null; then
                                echo 'MinIO is ready'
                                exit 0
                            fi
                            echo "Attempt \$i/15 — waiting 3 seconds..."
                            sleep 3
                        done
                        echo 'MinIO did not become ready in time'
                        exit 1
                    """
                }
            }
        }

        stage('Create Bucket') {
            steps {
                script {
                    echo "--- Creating bucket '${MINIO_BUCKET}' if not exists ---"
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'minio-credentials', 
                            usernameVariable: 'MINIO_USER',
                            passwordVariable: 'MINIO_PASSWORD'
                        )
                    ]) {
                        sh """
                            docker exec ${CONTAINER_NAME} mc alias set local \
                                http://localhost:9000 \
                                ${MINIO_USER} \
                                ${MINIO_PASSWORD}

                            docker exec ${CONTAINER_NAME} mc mb --ignore-existing \
                                local/${MINIO_BUCKET}

                            echo "Bucket '${MINIO_BUCKET}' is ready"
                        """
                    }
                }
            }
        }

        stage('Verify') {
            steps {
                script {
                    echo "--- Verifying MinIO status ---"
                    sh "docker ps --filter name=${CONTAINER_NAME} --format 'Container: {{.Names}} | Status: {{.Status}}'"
                    echo "--- MinIO console available at: http://<host>:${MINIO_UI_PORT} ---"
                    echo "--- MinIO API available at: http://<host>:${MINIO_API_PORT} ---"
                }
            }
        }
    }

    post {
        always {
            echo "--- Cleaning up workspace ---"
            deleteDir()
        }
        failure {
            echo "--- MinIO deployment FAILED. Check logs above. ---"
        }
    }
}
