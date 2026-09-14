pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPOSITORY = 'aws-ecr-docker-deployment'
        IMAGE_TAG = "${BUILD_NUMBER}"
        CONTAINER_NAME = 'aws-ecr-docker-deployment'
        APP_PORT = '5000'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'python3 -m pip install -r requirements.txt'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'python3 -m pytest --junitxml=test-results.xml'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                        -t ${ECR_REPOSITORY}:${IMAGE_TAG} \
                        .
                '''
            }
        }

        stage('Authenticate to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login \
                        --username AWS \
                        --password-stdin \
                        $(aws sts get-caller-identity --query Account --output text).dkr.ecr.${AWS_REGION}.amazonaws.com
                '''
            }
        }

        stage('Tag Docker Image') {
            steps {
                sh '''
                    ECR_URI=$(aws ecr describe-repositories \
                        --repository-names ${ECR_REPOSITORY} \
                        --region ${AWS_REGION} \
                        --query 'repositories[0].repositoryUri' \
                        --output text)

                    docker tag \
                        ${ECR_REPOSITORY}:${IMAGE_TAG} \
                        ${ECR_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    ECR_URI=$(aws ecr describe-repositories \
                        --repository-names ${ECR_REPOSITORY} \
                        --region ${AWS_REGION} \
                        --query 'repositories[0].repositoryUri' \
                        --output text)

                    docker push ${ECR_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy Container') {
            steps {
                sh '''
                    ECR_URI=$(aws ecr describe-repositories \
                        --repository-names ${ECR_REPOSITORY} \
                        --region ${AWS_REGION} \
                        --query 'repositories[0].repositoryUri' \
                        --output text)

                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true

                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${APP_PORT}:5000 \
                        ${ECR_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    sleep 5
                    curl --fail http://localhost:${APP_PORT}/health
                '''
            }
        }
    }

    post {
        always {
            junit 'test-results.xml'
        }

        success {
            echo 'ECR CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the stage logs for details.'
        }
    }
}
