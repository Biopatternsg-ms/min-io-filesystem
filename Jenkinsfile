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
                    echo "--- Attempting to connect Jenkins agent container to network '${NETWORK}' ---"
                    sh "docker network connect ${NETWORK} \$(hostname) || true"
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
                    sh """
                        GW_IP=\$(ip route | awk '/default/ {print \$3}' 2>/dev/null || true)
                        if [ -z "\$GW_IP" ]; then
                            GW_IP="localhost"
                        fi

                        ENDPOINTS="http://localhost:${MINIO_API_PORT} http://${CONTAINER_NAME}:9000 http://\${GW_IP}:${MINIO_API_PORT}"
                        WORKING_ENDPOINT=""

                        for i in \$(seq 1 20); do
                            for ep in \$ENDPOINTS; do
                                STATUS=\$(curl -s -o /dev/null -w "%{http_code}" \${ep}/minio/health/live || true)
                                if [ "\$STATUS" = "200" ]; then
                                    echo "MinIO is ready at \${ep} (HTTP 200)"
                                    WORKING_ENDPOINT="\${ep}"
                                    break 2
                                fi
                            done
                            echo "Attempt \$i/20 — MinIO not ready yet, waiting 3 seconds..."
                            sleep 3
                        done

                        if [ -z "\$WORKING_ENDPOINT" ]; then
                            echo "MinIO did not become ready in time. Printing container status and logs:"
                            docker ps -a --filter name=${CONTAINER_NAME}
                            docker logs ${CONTAINER_NAME} | tail -n 50
                            exit 1
                        fi

                        echo "\$WORKING_ENDPOINT" > .minio_endpoint
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
                            MINIO_ENDPOINT=\$(cat .minio_endpoint)
                            ${MC_BIN} alias set local \
                                "\${MINIO_ENDPOINT}" \
                                "\$MINIO_ROOT_USER" \
                                "\$MINIO_ROOT_PASSWORD"

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
