pipeline {
    agent any

    environment {
        APP_NAME = "nodeapp"
        DOCKERHUB_USER = "your-dockerhub-user"
        IMAGE_TAG = "v${BUILD_NUMBER}"
        DEPLOY_SERVER_DEV = "ubuntu@13.233.xx.xx"
        DEPLOY_SERVER_PROD = "ubuntu@13.232.xx.xx"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git 'https://github.com/your/repo.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('Lint') {
            steps {
                sh "npm run lint"
            }
        }

        stage('Run Tests') {
            steps {
                sh "npm test"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo $PASS | docker login -u $USER --password-stdin"
                    sh "docker push ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to Dev') {
            steps {
                sshagent(['ec2-ssh']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no $DEPLOY_SERVER_DEV '
                        docker pull ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG} &&
                        docker stop ${APP_NAME} || true &&
                        docker rm ${APP_NAME} || true &&
                        docker run -d -p 80:80 --name ${APP_NAME} ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}
                    '
                    """
                }
            }
        }

        stage('Manual Approval for Production') {
            steps {
                input message: "Deploy to Production?"
            }
        }

        stage('Blue/Green Deployment to Prod') {
            steps {
                sshagent(['ec2-ssh']) {
                    script {
                        active = sh(
                          script: "ssh $DEPLOY_SERVER_PROD \"docker ps --filter 'name=${APP_NAME}' --format '{{.Names}}'\"",
                          returnStdout: true
                        ).trim()

                        if (active == "${APP_NAME}_blue") {
                            next = "green"
                        } else {
                            next = "blue"
                        }

                        sh """
                        ssh $DEPLOY_SERVER_PROD '
                            docker pull ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG} &&
                            docker stop ${APP_NAME}_${next} || true &&
                            docker rm ${APP_NAME}_${next} || true &&
                            docker run -d -p 80:80 --name ${APP_NAME}_${next} ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}
                        '
                        """
                    }
                }
            }
        }
    }
}
