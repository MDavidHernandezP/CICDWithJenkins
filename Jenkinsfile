pipeline {
    agent any
    environment {
        PORT = ''
        IMAGE_NAME = ''
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
                    } else {
                        PORT = '3001'
                        IMAGE_NAME = 'nodedev:v1.0'
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
        stage('Clean Old Containers') {
            steps {
                script {
                    def containerId = sh(script: "docker ps -q --filter ancestor=${IMAGE_NAME}", returnStdout: true).trim()
                    if (containerId) {
                        sh "docker stop ${containerId}"
                        sh "docker rm ${containerId}"
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                sh "docker run -d -p ${PORT}:3000 ${IMAGE_NAME}"
            }
        }
    }
}
