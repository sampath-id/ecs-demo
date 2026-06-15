pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "236726878226"
        AWS_REGION     = "ap-south-1"
        ECR_REPO       = "app"
        LOCAL_IMAGE    = "app"
        ECR_IMAGE      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:latest"
    }

    stages {

        stage('Clone Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/sampath-id/ecs-demo.git',
                    credentialsId: 'sampath-id'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    script {
                        def scannerHome = tool 'sonar-scanner'

                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=my-app \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://13.235.216.241:9000 \
                        -Dsonar.login=${SONAR_AUTH_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${LOCAL_IMAGE} ."
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        aws sts get-caller-identity
                        aws ecr get-login-password --region $AWS_REGION | \
                        docker login --username AWS --password-stdin \
                        $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                    '''
                }
            }
        }

        stage('Tag Docker Image') {
            steps {
                sh "docker tag ${LOCAL_IMAGE} ${ECR_IMAGE}"
            }
        }

        stage('Push Image to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh "docker push ${ECR_IMAGE}"
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        aws ecs update-service \
                        --cluster ecs-cluster \
                        --service app-service \
                        --force-new-deployment \
                        --region $AWS_REGION
                    '''
                }
            }
        }

    } // End of stages

    post {
        success {
            echo 'Docker image pushed to Amazon ECR and ECS deployed successfully!'
        }

        failure {
            echo 'Pipeline failed!'
        }
    }

} // End of pipeline
