pipeline {
    agent any
    
    environment {
        DD_AGENT_CONTAINER = 'dd-agent'
        DD_ENV = 'prod'
        DD_SITE = 'datadoghq.com'
        
        APP_NAME = 'kursova-angular'
        DOCKERHUB_REPO = 'adovgoshiyari42/${APP_NAME}'
        SSH_KEY_ID = 'jenkins-github-key'
    }

    stages {
        stage('checkout') {
            steps {
            	checkout scm
            }
        stage('build docker image') {
            steps {
                script {
                    IMAGE_TAG = "${BUILD_NUMBER}"
                }
                
                sh """
                    docker build --cache-from ${DOCKERHUB_REPO}:latest \\
                    -t ${APP_NAME}:${IMAGE_TAG} -t ${APP_NAME}:latest .
                """

            }
        }
        stage('push docker image into docker hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "\${DOCKER_PASS}" | docker login -u "\$DOCKER_USER" --password-stdin
                        
                        docker tag ${APP_NAME}:${IMAGE_TAG} \$DOCKER_USER/${APP_NAME}:${IMAGE_TAG}
                        docker push \$DOCKER_USER/${APP_NAME}:${IMAGE_TAG}
                        
                        docker tag ${APP_NAME}:latest \$DOCKER_USER/${APP_NAME}:latest
                        docker push \$DOCKER_USER/${APP_NAME}:latest
                    """
                }
            }
        }
        stage('run datadog agent') {
            steps {
                withCredentials([string(credentialsId: 'datadog-api-key', variable: 'DD_API_KEY')]){
                    sh """
                        existing=\$(docker ps -aq -f "name=${DD_AGENT_CONTAINER}")
                        if [ ! -z "\$existing" ]; then
                            docker stop ${DD_AGENT_CONTAINER} || true
                            docker rm ${DD_AGENT_CONTAINER} || true
                        fi

                        docker run -d --name ${DD_AGENT_CONTAINER} \\
                            -e DD_API_KEY=$DD_API_KEY \\
                            -e DD_SITE=${DD_SITE} \\
                            -e DD_ENV=${DD_ENV} \\
                            -v /var/run/docker.sock:/var/run/docker.sock:ro \\
                            gcr.io/datadoghq/agent:7
                    """
                }
            }
        }
        stage('deploy container') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "\${DOCKER_PASS}" | docker login -u "\$DOCKER_USER" --password-stdin

                        existing=\$(docker ps -aq -f "name=${APP_NAME}")
                        if [ ! -z "\$existing" ]; then
                            docker stop ${APP_NAME} || true
                            docker rm ${APP_NAME} || true
                        fi
                            
                        docker pull \$DOCKER_USER/${APP_NAME}:${IMAGE_TAG}
                        docker run -d -p 80:80 --name ${APP_NAME} \$DOCKER_USER/${APP_NAME}:${IMAGE_TAG}
                """
                }
            }
        }
    }
    post {
        success { 
            echo 'is working!'
        }
        failure {
            echo "okay we went wrong"
        }
    }

