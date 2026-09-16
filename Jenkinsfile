pipeline {

    agent any

    environment {

        AWS_REGION = 'us-east-1'

        AWS_ACCOUNT_ID = sh(
            script: "aws sts get-caller-identity --query Account --output text",
            returnStdout: true
        ).trim()

        ECR_REPOSITORY = 'devops-platform'

        EKS_CLUSTER = 'devops-platform-eks'

        K8S_NAMESPACE = 'devops-demo'

        K8S_DEPLOYMENT = 'aws-java-app'

        CONTAINER_NAME = 'aws-java-app'

        IMAGE_TAG = "${BUILD_NUMBER}"

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        IMAGE_URI = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {

            steps {

                echo 'Checking out source code...'

                checkout scm
            }
        }


        stage('Environment Check') {

            steps {

                sh '''
                    echo "================================="
                    echo "Environment"
                    echo "================================="

                    java -version

                    mvn -version

                    docker --version

                    aws --version

                    kubectl version --client

                    aws sts get-caller-identity
                '''
            }
        }


        stage('Test') {

            steps {

                echo 'Running unit tests...'

                sh '''
                    mvn clean test
                '''
            }
        }


        stage('Build') {

            steps {

                echo 'Building Spring Boot application...'

                sh '''
                    mvn clean package -DskipTests
                '''
            }
        }


        stage('Docker Build') {

            steps {

                echo "Building Docker image: ${IMAGE_URI}"

                sh '''
                    docker build \
                      -t ${IMAGE_URI} \
                      -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest \
                      .
                '''
            }
        }


        stage('ECR Login') {

            steps {

                sh '''
                    aws ecr get-login-password \
                      --region ${AWS_REGION} \
                    | docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}
                '''
            }
        }


        stage('Push Image') {

            steps {

                echo 'Pushing Docker image to Amazon ECR...'

                sh '''
                    docker push ${IMAGE_URI}

                    docker push \
                      ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                '''
            }
        }


        stage('Configure Kubernetes') {

            steps {

              sh '''
            aws eks update-kubeconfig \
              --region ${AWS_REGION} \
              --name ${EKS_CLUSTER}

            kubectl cluster-info
        '''
    }
}


        stage('Deploy to Kubernetes') {

            steps {

             sh '''
            sed -i "s|IMAGE_PLACEHOLDER|${IMAGE_URI}|g" \
              kubernetes/deployment.yaml

            kubectl apply \
              -f kubernetes/namespace.yaml

            kubectl apply \
              -f kubernetes/deployment.yaml

            kubectl apply \
              -f kubernetes/service.yaml
        '''
    }
}


        stage('Rollout') {

             steps {

              sh '''
            kubectl rollout status \
              deployment/${K8S_DEPLOYMENT} \
              -n ${K8S_NAMESPACE} \
              --timeout=180s
        '''
    }
}


        stage('Verify') {

          steps {

            sh '''
            kubectl get pods \
              -n ${K8S_NAMESPACE}

            kubectl get deployment \
              -n ${K8S_NAMESPACE}

            kubectl get service \
              -n ${K8S_NAMESPACE}
        '''
    }
}


    post {

        success {

            echo '''
            ============================================
            CI/CD PIPELINE SUCCESSFUL
            ============================================
            '''
        }

        failure {

            echo '''
            ============================================
            CI/CD PIPELINE FAILED
            ============================================
            '''
        }

        always {

            sh '''
                docker image prune -f || true
            '''
        }
    }
}