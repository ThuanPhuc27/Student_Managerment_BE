pipeline {
    agent any 
    environment {
        DOCKER_IMAGE_NAME = "harbor.lptdevops.website/student_managerment/backend"
        DOCKER_IMAGE_TAG = "latest"
        REGISTRY_URL = "harbor.lptdevops.website"
        ECR_REGISTRY = '160885258086.dkr.ecr.ap-northeast-2.amazonaws.com'
        ECR_PROJECT = 'student_management_be'

        TASK_DEFINITION =""
        NEW_TASK_DEFINITION=""
        NEW_TASK_INFO=""
        NEW_REVISION=""
        TASK_FAMILY="backend-task-definition"
        CLUSTER_NAME="ecs-cluster"
        SERVICE_NAME="be-service"
        REGION="ap-northeast-2"
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

        stage('Update task definition and force deploy ecs service') {
            steps {
                script {
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            env.userChoice = input message: 'Do you want to create new task definition',
                            parameters: [
                                choice(name: 'Versioning Service', choices: ['no', 'yes'], description: 'Choose "yes" if you want to create!')
                            ]
                        }

                        if (env.userChoice == 'yes') {
                            sh '''
                                apt update && apt install jq -y

                                TASK_DEFINITION=$(aws ecs describe-task-definition --task-definition ${TASK_FAMILY} --region ${REGION})

                                NEW_TASK_DEFINITION=$(echo $TASK_DEFINITION | jq --arg IMAGE "${ECR_REGISTRY}/${ECR_PROJECT}:${DOCKER_IMAGE_TAG}" '.taskDefinition | .containerDefinitions[0].image = $IMAGE | del(.taskDefinitionArn) | del(.revision) | del(.status) | del(.requiresAttributes) | del(.compatibilities) |  del(.registeredAt)  | del(.registeredBy)')

                                NEW_TASK_INFO=$(aws ecs register-task-definition --region ${REGION} --cli-input-json "$NEW_TASK_DEFINITION")
   
                                NEW_REVISION=$(echo $NEW_TASK_INFO | jq '.taskDefinition.revision')
                                
                                aws ecs update-service --cluster ${CLUSTER_NAME} --service ${SERVICE_NAME} --task-definition ${TASK_FAMILY}:${NEW_REVISION} --force-new-deployment
                            '''
                        } else {
                            echo "create cancelled."
                        }

                    } catch (Exception err) {
                        def user = err.getCauses()[0]?.getUser()
                        if (user == null || 'SYSTEM' == user.toString()) {
                            echo "Timeout. create cancelled."
                        } else {
                            echo "create cancelled by: ${user}"
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