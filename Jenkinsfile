pipeline {
    agent any

    environment {
        CONTAINER_NAME  = 'minio-filesystem'
        MINIO_API_PORT  = '10000'
        MINIO_UI_PORT   = '10001'
        MINIO_BUCKET    = 'biopatternsg-kb'
        NETWORK         = 'general-network'
        // mc binario descargado en el workspace del agente Jenkins
        MC_BIN          = './mc'
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
                            usernameVariable: 'MINIO_ROOT_USER',
                            passwordVariable: 'MINIO_ROOT_PASSWORD'
                        )
                    ]) {
                        sh 'docker compose up -d'
                    }
                    echo "--- MinIO container started ---"
                }
            }
        }

        stage('Wait for MinIO Ready') {
            steps {
                script {
                    echo "--- Waiting for MinIO to be ready (health endpoint) ---"
                    // Usa el health endpoint HTTP de MinIO, que no requiere mc ni credenciales.
                    // El puerto 10000 es el mapeado al host; dentro del contenedor sigue siendo 9000.
                    sh """
                        for i in \$(seq 1 20); do
                            STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${MINIO_API_PORT}/minio/health/live)
                            if [ "\$STATUS" = "200" ]; then
                                echo "MinIO is ready (HTTP 200)"
                                exit 0
                            fi
                            echo "Attempt \$i/20 — status=\${STATUS}, waiting 3 seconds..."
                            sleep 3
                        done
                        echo "MinIO did not become ready in time"
                        exit 1
                    """
                }
            }
        }

        stage('Setup mc Client') {
            steps {
                script {
                    echo "--- Downloading mc (MinIO Client) to Jenkins agent ---"
                    // Descarga mc en el workspace del agente Jenkins.
                    // No requiere que mc esté instalado en la imagen del contenedor.
                    sh """
                        curl -sSf https://dl.min.io/client/mc/release/linux-amd64/mc -o ${MC_BIN}
                        chmod +x ${MC_BIN}
                        ${MC_BIN} --version
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
                            usernameVariable: 'MINIO_ROOT_USER',
                            passwordVariable: 'MINIO_ROOT_PASSWORD'
                        )
                    ]) {
                        sh """
                            ${MC_BIN} alias set local \
                                http://localhost:${MINIO_API_PORT} \
                                ${MINIO_ROOT_USER} \
                                ${MINIO_ROOT_PASSWORD}

                            ${MC_BIN} mb --ignore-existing local/${MINIO_BUCKET}

                            echo "Bucket '${MINIO_BUCKET}' is ready"
                            ${MC_BIN} ls local/
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
