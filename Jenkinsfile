pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = "236726878226"
        AWS_REGION     = "ap-south-1"        // ✅ fixed region
        ECR_REPO       = "app"
        LOCAL_IMAGE    = "my-app:latest"
        ECR_IMAGE      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:latest"
    }
    stages {

        stage('Clone Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/sampath-id/docker-hub.git',
                    credentialsId: 'sampath-id'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    script {
                        def scannerHome = tool 'sonar-scanner'  // ✅ fixed
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=my-app \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://13.200.200.62:9000 \
                            -Dsonar.login=${SONAR_AUTH_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {  // ✅ added timeout
                    waitForQualityGate abortPipeline: true
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
                    $class           : 'AmazonWebServicesCredentialsBinding',
                    credentialsId    : 'aws-creds',
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
                    $class           : 'AmazonWebServicesCredentialsBinding',
                    credentialsId    : 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh "docker push ${ECR_IMAGE}"
                }
            }
        }
    }

    post {
        success {
            echo 'Docker image pushed to Amazon ECR successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
