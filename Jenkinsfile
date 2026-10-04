pipeline {
    agent any

    environment {
        TAG         = "latest"

        IMAGE_WEB   = "israelsalvador/web:${TAG}"
        IMAGE_DB    = "israelsalvador/db:${TAG}"
        IMAGE_NGINX = "israelsalvador/nginx:${TAG}"

        COMPOSE_FILE = "${env.COMPOSE_FILE}"
    }

    stages {

        stage('Checkout') {
            steps {
                sh 'echo "Checking out repository..."'

                checkout scm

                slackSend(
                    channel: '#ci-devops',
                    message: "Checkout iniciado",
                    tokenCredentialId: 'slack-token'
                )
            }
        }

        stage('Build Images') {
            steps {

                slackSend(
                    channel: '#ci-devops',
                    message: "Build das imagens iniciado",
                    tokenCredentialId: 'slack-token'
                )

                script {

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub',
                            usernameVariable: 'DOCKERHUB_USER',
                            passwordVariable: 'DOCKERHUB_TOKEN'
                        )
                    ]) {

                        sh '''
                            echo "$DOCKERHUB_TOKEN" | docker login \
                                -u "$DOCKERHUB_USER" \
                                --password-stdin

                            echo "Build da imagem WEB..."
                            docker build \
                                -t israelsalvador/web:latest \
                                -f Dockerfileweb .

                            echo "Build da imagem DB..."
                            docker build \
                                -t israelsalvador/db:latest \
                                -f Dockerfiledb .

                            echo "Build da imagem NGINX..."
                            docker build \
                                -t israelsalvador/nginx:latest \
                                -f Dockerfilenginx .

                            echo "Push da imagem WEB..."
                            docker push israelsalvador/web:latest

                            echo "Push da imagem DB..."
                            docker push israelsalvador/db:latest

                            echo "Push da imagem NGINX..."
                            docker push israelsalvador/webserver-golang:latest

                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Security Scan (Trivy)') {
            steps {

                slackSend(
                    channel: '#ci-devops',
                    message: "Rodando Trivy Scan...",
                    tokenCredentialId: 'slack-token'
                )

                sh """
                    trivy image ${IMAGE_WEB} \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format json \
                        --output trivy-web.json

                    trivy image ${IMAGE_DB} \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format json \
                        --output trivy-db.json

                    trivy image ${IMAGE_NGINX} \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format json \
                        --output trivy-nginx.json
                """
            }

            post {
                always {

                    archiveArtifacts(
                        artifacts: 'trivy-*.json',
                        fingerprint: true
                    )

                    slackSend(
                        channel: '#ci-devops',
                        message: "Trivy scan concluído. Relatórios disponíveis nos artefatos.",
                        tokenCredentialId: 'slack-token'
                    )
                }
            }
        }

        stage('Test in Containers') {
            steps {

                slackSend(
                    channel: '#ci-devops',
                    message: "Subindo containers para testes...",
                    tokenCredentialId: 'slack-token'
                )

                sh """
                    docker rm -f web1 web2 web3 db nginx || true

                    docker compose -f ${COMPOSE_FILE} up -d

                    sleep 5

                    curl -I http://localhost || exit 1

                    docker compose -f ${COMPOSE_FILE} down
                """
            }
        }

        stage('Deploy to Production') {
            steps {

                slackSend(
                    channel: '#ci-devops',
                    message: "Deploy em produção iniciado...",
                    tokenCredentialId: 'slack-token'
                )

                sh """
                    docker compose -f ${COMPOSE_FILE} pull

                    docker compose -f ${COMPOSE_FILE} up -d
                """
            }
        }
    }

    post {

        success {
            slackSend(
                channel: '#ci-devops',
                message: "Pipeline finalizado com sucesso! Build #${BUILD_NUMBER}",
                tokenCredentialId: 'slack-token'
            )
        }

        failure {
            slackSend(
                channel: '#ci-devops',
                message: "Pipeline falhou! Verifique o console. Build #${BUILD_NUMBER}",
                tokenCredentialId: 'slack-token'
            )
        }
    }
}
