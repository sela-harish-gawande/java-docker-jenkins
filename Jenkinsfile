pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent'
        }
    }

    environment {
        ACCOUNT_ID       = '378898159802'
        AWS_REGION       = 'ap-south-1'
        ECR_REPO_NAME    = 'jenkins'
        IMAGE_TAG        = "${BUILD_NUMBER}"
        FULL_IMAGE       = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest"
        AWS_CREDENTIALS  = credentials('aws-credentials')
        SONARQUBE_SERVER = 'sonarqube'
        SONAR_SCANNER    = 'sonar-scanner'
    }

    stages {

        stage('Initialize') {
            steps {
                echo "🔧 Initializing Build Environment"
                script {
                    sh '''
                    mkdir -p /run/buildkit
                    buildkitd --root /run/buildkit &
                    sleep 2
                    '''
                }
            }
        }

        stage('Checkout Source') {
            steps {
                echo "📦 Checking out source code..."
                checkout scm
            }
        }

        stage('Preliminary Scan: Trivy Filesystem') {
            steps {
                echo "🔍 Running Trivy filesystem scan..."
                sh '''
                trivy fs . || echo "Trivy FS scan completed with warnings"
                '''
            }
        }

        stage('Static Code Analysis: SonarQube') {
            steps {
                echo "🧠 Running SonarQube Analysis..."
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    withEnv(["PATH+SONAR=${tool SONAR_SCANNER}/bin"]) {
                        sh '''
                        set -euxo pipefail

                        if [ -d "target/classes" ]; then
                            EXTRA_FLAGS="-Dsonar.java.binaries=**/target/classes"
                        else
                            EXTRA_FLAGS="-Dsonar.exclusions=**/*.java,**/node_modules/**,**/dist/**,**/target/**,**/.git/**"
                        fi

                        sonar-scanner \
                            -Dsonar.projectKey=plumhq-jenkins-sample \
                            -Dsonar.projectName="PlumHQ Jenkins Sample" \
                            -Dsonar.sources=. \
                            $EXTRA_FLAGS \
                            -Dsonar.host.url=$SONAR_HOST_URL \
                            -Dsonar.token=$SONAR_AUTH_TOKEN
                        '''
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                echo "✅ Waiting for Quality Gate result..."
                timeout(time: 20, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                echo "🐳 Building Docker Image..."
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                    export XDG_RUNTIME_DIR=/run/user/$(id -u)
                    mkdir -p $XDG_RUNTIME_DIR
                    chown $(id -u):$(id -g) $XDG_RUNTIME_DIR

                    aws sts get-caller-identity
                    aws ecr get-login-password --region $AWS_REGION | nerdctl login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

                    nerdctl build -t $ECR_REPO_NAME:$IMAGE_TAG .
                    nerdctl tag $ECR_REPO_NAME:$IMAGE_TAG $FULL_IMAGE
                    '''
                }
            }
        }

        stage('Container Image Scan: Trivy') {
            steps {
                echo "🔎 Scanning Docker image with Trivy..."
                sh '''
                trivy image --severity HIGH,CRITICAL --exit-code 1 $FULL_IMAGE
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                echo "📤 Pushing Docker image to AWS ECR..."
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                    aws ecr get-login-password --region $AWS_REGION | nerdctl login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                    nerdctl push $FULL_IMAGE

                    echo "🧹 Cleaning up local Docker images..."
                    nerdctl images --format '{{.Repository}} {{.ID}}' | grep jenkins | awk '{print $2}' | xargs -r nerdctl rmi -f
                    '''
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            environment {
                TARGET_REPO = 'github.com/vikasb-sela/ArgoCD.git'
                PATCH_FILE  = 'k8s-manifests/notifications-service/overlays/stg/patch.yaml'
            }
            steps {
                echo "📝 Updating Kubernetes deployment manifest..."
                withCredentials([string(credentialsId: 'target-github', variable: 'GITHUB_PAT')]) {
                    sh '''
                    git config --global user.name "jenkins-bot"
                    git config --global user.email "jenkins@local"

                    rm -rf ArgoCD
                    git clone https://${GITHUB_PAT}@${TARGET_REPO}
                    cd ArgoCD

                    git pull origin main --rebase
                    sed -i "s|image: .*|image: ${FULL_IMAGE}|" ${PATCH_FILE}

                    git add ${PATCH_FILE}
                    git commit -m "ci: update image to ${FULL_IMAGE}" || echo "No changes to commit"
                    git push origin main
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "📌 Pipeline execution completed"
        }
        failure {
            echo "❌ Pipeline failed. Check above logs for error details."
        }
        success {
            echo "🎉 Pipeline succeeded!"
        }
    }
}
