pipeline {
    agent any 
    environment {
        DOCKER_IMAGE_NAME = "harbor.lptdevops.website/student_managerment/backend"
        DOCKER_IMAGE_TAG = "latest"
        REGISTRY_URL = "harbor.lptdevops.website"
        ECR_REGISTRY = '160885258086.dkr.ecr.ap-northeast-2.amazonaws.com'
        ECR_PROJECT = 'student_management_be'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ."
                }
            }
        }

        stage('Login to Docker Registry') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'harbor', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo $DOCKER_PASS | docker login ${REGISTRY_URL} -u $DOCKER_USER --password-stdin"
                    }
                    sh 'aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 160885258086.dkr.ecr.ap-northeast-2.amazonaws.com'
                }
            }
        }

        stage('Push Docker Image & Upload to ECR') {
            parallel {
                stage('Push Docker Image to Harbor') {
                    steps {
                        script {
                            sh "docker push ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                        }
                    }
                }

                stage('Upload image to ECR') {
                    steps {
                        script {
                            try {
                                timeout(time: 5, unit: 'MINUTES') {
                                    env.userChoice = input message: 'Do you want to push to ecr',
                                    parameters: [
                                        choice(name: 'Versioning Service', choices: 'no\nyes', description: 'Choose "yes" if you want to push!')
                                    ]
                                }

                                if (env.userChoice == 'yes') {
                                    echo "Push to ECR started..."
                                    sh 'docker tag ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ${ECR_REGISTRY}/${ECR_PROJECT}:${DOCKER_IMAGE_TAG}'
                                    sh 'docker push ${ECR_REGISTRY}/${ECR_PROJECT}:${DOCKER_IMAGE_TAG}'
                                    echo "Push success!"
                                } else {
                                    echo "Push cancelled."
                                }
                            } catch (Exception err) {
                                def user = err.getCauses()[0]?.getUser()
                                if (user == null || 'SYSTEM' == user.toString()) {
                                    echo "Timeout. Push cancelled."
                                } else {
                                    echo "Push cancelled by: ${user}"
                                }
                            }
                        }
                    }
                }
            }
        }


    }
    post {
        always {
            sh "docker logout ${REGISTRY_URL}"
        }
        success {
            echo "Docker image ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} pushed successfully!"
        }
        failure {
            echo "Build or push failed. Please check the logs."
        }
    }
}