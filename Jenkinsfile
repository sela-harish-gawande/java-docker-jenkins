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

        // SonarQube bits (must match Jenkins config)
        SONARQUBE_SERVER = 'sonarqube'      // Manage Jenkins → System → SonarQube servers
        SONAR_SCANNER    = 'sonar-scanner'  // Global Tool Configuration → SonarQube Scanner
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

        /* ---- ADD: SonarQube analysis (no sonar-project.properties needed) ---- */
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

        /* ---- your original stages remain unchanged below ---- */
        stage('Build Docker Image') {
            steps {
                script {
                    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
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
                        aws ecr get-login-password --region ap-south-1 | nerdctl login --username AWS --password-stdin $Account_ID.dkr.ecr.ap-south-1.amazonaws.com
                        nerdctl push $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
                        '''
                    }
                }
            }
        }
    }
}
