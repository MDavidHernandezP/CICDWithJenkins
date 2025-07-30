pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'tronikode/nodeapp-jenkins'
        IMAGE_TAG = 'v1.0'
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
                    def branch = env.BRANCH_NAME ?: env.GIT_BRANCH?.tokenize('/')[-1]
                    
                    if (branch == 'main') {
                        env.PORT = '3000'
                        env.IMAGE_NAME = "nodemain:${IMAGE_TAG}"
                        env.CONTAINER_NAME = 'container-main'
                    } else {
                        env.PORT = '3001'
                        env.IMAGE_NAME = "nodedev:${IMAGE_TAG}"
                        env.CONTAINER_NAME = 'container-dev'
                    }
                    echo "Configured for branch: ${branch} -> port: ${env.PORT}, image: ${env.IMAGE_NAME}, container: ${env.CONTAINER_NAME}"
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
                sh "docker build -t ${env.IMAGE_NAME} ."
            }
        }

        stage('Tag & Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        def remoteImage = "${env.DOCKERHUB_REPO}:${env.IMAGE_NAME.split(':')[1]}"
                        echo "Tagging image as ${remoteImage}"

                        sh """
                            echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                            docker tag ${env.IMAGE_NAME} ${remoteImage}
                            docker push ${remoteImage}
                            docker logout
                        """
                    }
                }
            }
        }

        stage('Clean Old Container for Env') {
            steps {
                script {
                    def containerOnPort = sh(script: "docker ps --format '{{.ID}} {{.Ports}}' | grep '${env.PORT}->' | awk '{print \$1}'", returnStdout: true).trim()
                    if (containerOnPort) {
                        echo "Stopping container using port ${env.PORT}"
                        sh "docker stop ${containerOnPort} || true"
                        sh "docker rm ${containerOnPort} || true"
                    }

                    def existingByName = sh(script: "docker ps -a -q --filter name=${env.CONTAINER_NAME}", returnStdout: true).trim()
                    if (existingByName) {
                        echo "Removing old container named ${env.CONTAINER_NAME}"
                        sh "docker stop ${env.CONTAINER_NAME} || true"
                        sh "docker rm ${env.CONTAINER_NAME} || true"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh "docker run -d --name ${env.CONTAINER_NAME} -p ${env.PORT}:3000 ${env.IMAGE_NAME}"
            }
        }

        stage('Trigger Deploy Job') {
            steps {
                script {
                    def branch = env.BRANCH_NAME ?: env.GIT_BRANCH?.tokenize('/')[-1]
                    if (branch == 'main') {
                        build job: 'Deploy_to_main', wait: false
                    } else if (branch == 'dev') {
                        build job: 'Deploy_to_dev', wait: false
                    } else {
                        echo "No downstream deploy job triggered for branch: ${branch}"
                    }
                }
            }
        }
    }
}
