pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent'
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
                    nerdctl images --format '{{.Repository}} {{.ID}}' | grep jenkins | awk '{print $2}' | xargs -r nerdctl rmi -f
                    nerdctl pull $Account_ID.dkr.ecr.ap-south-1.amazonaws.com/jenkins || echo "Pull failed"
                    nerdctl images
                    '''
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    withEnv(["PATH+SONAR=${tool SONAR_SCANNER}/bin"]) {
                        sh '''
                          set -euxo pipefail

                          # If Maven produced classes, point Java sensor to them; else exclude .java to avoid failure
                          if [ -d "target/classes" ]; then
                            EXTRA_FLAGS="-Dsonar.java.binaries=**/target/classes"
                          else
                            EXTRA_FLAGS="-Dsonar.exclusions=**/*.java,**/node_modules/**,**/dist/**,**/target/**,**/.git/**"
                          fi

                          sonar-scanner \
                            -Dsonar.projectKey=plumhq-jenkins-sample \
                            -Dsonar.projectName="PlumHQ Jenkins Sample" \
                            -Dsonar.sources=. \
                            ${EXTRA_FLAGS} \
                            -Dsonar.host.url=$SONAR_HOST_URL \
                            -Dsonar.token=$SONAR_AUTH_TOKEN

                          # For coverage later, add for JaCoCo:
                          # -Dsonar.java.coveragePlugin=jacoco \
                          # -Dsonar.coverage.jacoco.xmlReportPaths=**/target/site/jacoco/jacoco.xml
                        '''
                    }
                }
            }
        }

        /* ---- ADD: enforce Quality Gate ---- */
        stage('Quality Gate') {
            steps {
                timeout(time: 20, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
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
                PATCH_FILE = 'k8s-manifests/notifications-service/overlays/stg/patch.yaml'
                FULL_IMAGE = "${Account_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest"
            }
            steps {
                withCredentials([string(credentialsId: 'target-github', variable: 'GITHUB_PAT')]) {
                    sh '''
                        echo "Cloning target repo..."
                        git config --global user.name "jenkins-bot"
                        git config --global user.email "jenkins@local"
                        rm -rf ArgoCD
                        git clone https://${GITHUB_PAT}@${TARGET_REPO}
                        cd ArgoCD
                        git pull origin main --rebase
                        ls -la
                        echo "Updating image in patch file..."
                        sed -i "s|image: .*|image: ${FULL_IMAGE}|" ${PATCH_FILE}
                        echo "Committing and pushing changes..."
                        git diff ${PATCH_FILE}
                        git add ${PATCH_FILE}
                        git commit -m "ci: update image to ${FULL_IMAGE}" || echo "No changes to commit"
                        git push origin main
                    '''
                }
            }
        }
    }
}
