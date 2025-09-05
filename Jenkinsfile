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
        AWS_CREDENTIALS = credentials('aws-credentials')  // AWS credential ID in Jenkins
    }
    stages {
        stage('Checkout') {
            steps {
                echo "Checkout Done"
                script {
                    sh '''
                    trivy fs .
                    echo "Source code filesystem scanned with Trivy"
                    mkdir -p /run/buildkit
                    buildkitd --root /run/buildkit &
                    sleep 2
                    nerdctl pull $Account_ID.dkr.ecr.ap-south-1.amazonaws.com/jenkins || echo "Pull failed"
                    nerdctl images
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh '''
                        export XDG_RUNTIME_DIR=/run/user/$(id -u)
                        mkdir -p $XDG_RUNTIME_DIR
                        chown $(id -u):$(id -g) $XDG_RUNTIME_DIR
                        aws sts get-caller-identity
                        aws ecr get-login-password --region $AWS_REGION | nerdctl login --username AWS --password-stdin $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                        nerdctl build -t $ECR_REPO_NAME:$IMAGE_TAG .
                        nerdctl tag $ECR_REPO_NAME:$IMAGE_TAG $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image --severity HIGH,CRITICAL --exit-code 1 $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
                '''
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
                        aws ecr get-login-password --region $AWS_REGION | nerdctl login --username AWS --password-stdin $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                        nerdctl push $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
                        '''
                    }
                }
            }
        }

        stage('Update Deployment Manifest in k8s-manifests') {
            environment {
                TARGET_REPO = 'github.com/vikasb-sela/ArgoCD.git'
                PATCH_FILE = 'ArgoCD/k8s-manifests/notifications-service/overlays/stg/patch.yaml'
                FULL_IMAGE = "${Account_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest"
            }
            steps {
                withCredentials([string(credentialsId: 'target-github', variable: 'GITHUB_PAT')]) {
                    sh '''
                        echo "Cloning target repo..."
                        git config --global user.name "jenkins-bot"
                        git config --global user.email "jenkins@local"
                        git clone https://${GITHUB_PAT}@${TARGET_REPO}

                        echo "Updating image in patch file..."
                        sed -i "s|image: .*|image: ${FULL_IMAGE}|" ${PATCH_FILE}

                        echo "Committing and pushing changes..."
                        git add ${PATCH_FILE}
                        git commit -m "ci: update image to ${FULL_IMAGE}" || echo "No changes to commit"
                        git push https://${GITHUB_PAT}@${TARGET_REPO} HEAD:main
                    '''
                }
            }
        }
    }
}
