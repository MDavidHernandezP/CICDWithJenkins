pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'tronikode/nodeapp-jenkins'
        VERSION = 'v1.0'
        PORT = ''
        IMAGE_NAME = ''
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
                    if (env.BRANCH_NAME == 'main') {
                        PORT = '3000'
                        IMAGE_NAME = "nodemain:${VERSION}"
                        CONTAINER_NAME = 'container-main'
                    } else if (env.BRANCH_NAME == 'dev') {
                        PORT = '3001'
                        IMAGE_NAME = "nodedev:${VERSION}"
                        CONTAINER_NAME = 'container-dev'
                    } else {
                        error "Branch ${env.BRANCH_NAME} is not supported for this pipeline."
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
                script {
                    sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_NAME} ."
                }
            }
        }

        stage('Tag & Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        '''
                        sh "docker push ${DOCKERHUB_REPO}:${IMAGE_NAME}"
                        sh "docker logout"
                    }
                }
            }
        }

        stage('Clean Old Container') {
            steps {
                script {
                    def containerOnPort = sh(script: "docker ps --format '{{.ID}} {{.Ports}}' | grep '${PORT}->' | awk '{print \$1}'", returnStdout: true).trim()
                    if (containerOnPort) {
                        echo "Stopping container using port ${PORT}"
                        sh "docker stop ${containerOnPort} || true"
                        sh "docker rm ${containerOnPort} || true"
                    }
                    
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
                sh "docker run -d --name ${CONTAINER_NAME} -p ${PORT}:3000 ${DOCKERHUB_REPO}:${IMAGE_NAME}"
            }
        }

        stage('Trigger Deploy Job') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        build job: 'Deploy_to_main', wait: false
                    } else if (env.BRANCH_NAME == 'dev') {
                        build job: 'Deploy_to_dev', wait: false
                    } else {
                        echo "No deploy job triggered for branch ${env.BRANCH_NAME}"
                    }
                }
            }
        }
    }
}
