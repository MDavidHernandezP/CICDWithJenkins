pipeline {
    agent any
    environment {
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
                script {
                    sh "docker build -t ${IMAGE_NAME} ."
                }
            }
        }
        stage('Clean Old Container for Env') {
            steps {
                script {
                    // Stop any container using the same port
                    def containerOnPort = sh(script: "docker ps --format '{{.ID}} {{.Ports}}' | grep '${PORT}->' | awk '{print \$1}'", returnStdout: true).trim()
                    if (containerOnPort) {
                        echo "Stopping container using port ${PORT}"
                        sh "docker stop ${containerOnPort} || true"
                        sh "docker rm ${containerOnPort} || true"
                    }
        
                    // Stop/remove container by name (in case it's lingering)
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
                sh "docker run -d --name ${CONTAINER_NAME} -p ${PORT}:3000 ${IMAGE_NAME}"
            }
        }
    }
}
