pipeline {
    agent {
        kubernetes {
            label 'jenkins-jenkins-agent'
        }
    }

    environment {
        Account_ID = '378898159802'
        AWS_REGION = 'ap-south-1'
        ECR_REPO_NAME = 'jenkins'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        AWS_CREDENTIALS = credentials('aws-credentials')  // Jenkins credential ID
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checkout Done"
                //git branch: 'main', url: 'https://github.com/sela-harish-gawande/java-docker-build-tutorial.git'
                script {
                    sh '''
                    echo "Welcome"
                    mkdir -p /run/buildkit
                    buildkitd --root /run/buildkit &
                    sleep 2 
                    nerdctl pull 378898159802.dkr.ecr.ap-south-1.amazonaws.com/jenkins || echo "Pull failed"
                    nerdctl images
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'jenkins-aws-cred', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh '''
                        # Login to ECR using AWS CLI and nerdctl
                        export XDG_RUNTIME_DIR=/run/user/$(id -u)
                        mkdir -p $XDG_RUNTIME_DIR
                        chown $(id -u):$(id -g) $XDG_RUNTIME_DIR
                        aws sts get-caller-identity
                        aws ecr get-login-password --region $AWS_REGION | nerdctl login --username AWS --password-stdin $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                        ls -la
                        nerdctl build -t $ECR_REPO_NAME:$IMAGE_TAG .
                        nerdctl tag $ECR_REPO_NAME:$IMAGE_TAG $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
                        '''
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                        export AWS_DEFAULT_REGION=$AWS_REGION
                        aws sts get-caller-identity
                        aws configure
                        aws ecr get-login-password --region ap-south-1 | nerdctl login --username AWS --password-stdin 378898159802.dkr.ecr.ap-south-1.amazonaws.com
                        nerdctl push $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
                        '''
                    }
                }
            }
        }
    }
}
