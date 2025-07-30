pipeline {
    agent any

    environment {
        PORT = ''
        IMAGE_NAME = ''
        DOCKERHUB_REPO = 'tronikode/nodeapp-jenkins'
        CONTAINER_NAME = ''
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Set Variables') {
            steps {
                script {
                    // Define port, image tag, and container name based on branch
                    if (env.BRANCH_NAME == 'main' || env.GIT_BRANCH == 'main') {
                        PORT = '3000'
                        IMAGE_NAME = 'nodemain:v1.0'
                        CONTAINER_NAME = 'container-main'
                    } else {
                        PORT = '3001'
                        IMAGE_NAME = 'nodedev:v1.0'
                        CONTAINER_NAME = 'container-dev'
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Tag & Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        def tag = IMAGE_NAME.split(':')[1]
                        def remoteImage = "${DOCKERHUB_REPO}:${tag}"

                        // Login to Docker Hub, tag and push the image
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        '''
                        sh "docker tag ${IMAGE_NAME} ${remoteImage}"
                        sh "docker push ${remoteImage}"
                        sh "docker logout"
                    }
                }
            }
        }

        stage('Clean Old Container for Env') {
            steps {
                script {
                    // Stop any container using the target port
                    def containerOnPort = sh(script: "docker ps --format '{{.ID}} {{.Ports}}' | grep '${PORT}->' | awk '{print \$1}'", returnStdout: true).trim()
                    if (containerOnPort) {
                        echo "Stopping container using port ${PORT}"
                        sh "docker stop ${containerOnPort} || true"
                        sh "docker rm ${containerOnPort} || true"
                    }

                    // Stop any old container with the same name
                    def existingByName = sh(script: "docker ps -a -q --filter name=${CONTAINER_NAME}", returnStdout: true).trim()
                    if (existingByName) {
                        echo "Stopping/removing container named ${CONTAINER_NAME}"
                        sh "docker stop ${CONTAINER_NAME} || true"
                        sh "docker rm ${CONTAINER_NAME} || true"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                // Run the container locally
                sh "docker run -d --name ${CONTAINER_NAME} -p ${PORT}:3000 ${IMAGE_NAME}"
            }
        }

        stage('Trigger Deploy Job') {
            steps {
                script {
                    // At the end, trigger another pipeline to pull the image from Docker Hub
                    if (env.BRANCH_NAME == 'main' || env.GIT_BRANCH == 'main') {
                        build job: 'Deploy_to_main', wait: false
                    } else if (env.BRANCH_NAME == 'dev' || env.GIT_BRANCH == 'dev') {
                        build job: 'Deploy_to_dev', wait: false
                    } else {
                        echo "No deploy job triggered for branch ${env.BRANCH_NAME}"
                    }
                }
            }
        }
    }
}
