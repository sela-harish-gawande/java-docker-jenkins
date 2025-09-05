pipeline {
  agent {
    kubernetes {
      label 'jenkins-jenkins-agent'
    }
  }

  environment {
    Account_ID      = '378898159802'
    AWS_REGION      = 'ap-south-1'
    ECR_REPO_NAME   = 'jenkins'
    IMAGE_TAG       = "${env.BUILD_NUMBER}"
    AWS_CREDENTIALS = credentials('aws-credentials') // Jenkins credential ID

    // SonarQube (must match your Jenkins config)
    SONARQUBE_SERVER = 'sonarqube'      // Manage Jenkins → System → SonarQube servers
    SONAR_SCANNER    = 'sonar-scanner'  // Global Tool Configuration → SonarQube Scanner
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        echo "Checkout Done"
      }
    }

    /* ---------- Compile Java for SonarQube (produces target/classes) ---------- */
    stage('Build & Test (Maven for Sonar)') {
      when { anyOf { expression { fileExists('pom.xml') }; expression { fileExists('mvnw') } } }
      steps {
        sh '''
          set -euxo pipefail
          if [ -x "./mvnw" ]; then
            ./mvnw -B clean verify -Dmaven.compiler.release=17
          else
            mvn -B clean verify -Dmaven.compiler.release=17
          fi
        '''
      }
    }

    /* ---------- SonarQube: analyze sources & enforce Quality Gate ---------- */
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          withEnv(["PATH+SONAR=${tool SONAR_SCANNER}/bin"]) {
            sh '''
              set -euxo pipefail

              # If Maven ran, point Java sensor to compiled classes; else exclude .java to avoid failure
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

              # If you generate coverage reports, add (examples):
              # -Dsonar.java.coveragePlugin=jacoco \
              # -Dsonar.coverage.jacoco.xmlReportPaths=**/target/site/jacoco/jacoco.xml
            '''
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    /* ---------- Trivy FS scan (source) ---------- */
    stage('Trivy FS Scan') {
      steps {
        sh '''
          set -euxo pipefail
          trivy fs .
          echo "Source code filesystem scanned with Trivy"
        '''
      }
    }

    /* ---------- BuildKit prep (start after Sonar to reduce memory pressure) ---------- */
    stage('BuildKit daemon (prep)') {
      steps {
        sh '''
          set +e
          mkdir -p /run/buildkit
          (buildkitd --root /run/buildkit >/dev/null 2>&1 &) || true
          sleep 2
          nerdctl images || true
        '''
      }
    }

    /* ---------- Build Docker image ---------- */
    stage('Build Docker Image') {
      steps {
        script {
          withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                               credentialsId: 'aws-credentials',
                               secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
            sh '''
              set -euxo pipefail
              export XDG_RUNTIME_DIR=/run/user/$(id -u)
              mkdir -p $XDG_RUNTIME_DIR
              chown $(id -u):$(id -g) $XDG_RUNTIME_DIR

              aws sts get-caller-identity
              aws ecr get-login-password --region $AWS_REGION \
                | nerdctl login --username AWS --password-stdin $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com

              ls -la
              nerdctl build -t $ECR_REPO_NAME:$IMAGE_TAG .
              nerdctl tag  $ECR_REPO_NAME:$IMAGE_TAG $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
            '''
          }
        }
      }
    }

    /* ---------- Trivy image scan (fail on HIGH/CRITICAL) ---------- */
    stage('Trivy Image Scan') {
      steps {
        sh '''
          set -euxo pipefail
          trivy image --severity HIGH,CRITICAL --exit-code 1 \
            $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
        '''
      }
    }

    /* ---------- Push to ECR ---------- */
    stage('Push to ECR') {
      steps {
        script {
          withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                               credentialsId: 'aws-credentials',
                               secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
            sh '''
              set -euxo pipefail
              export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
              export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
              export AWS_DEFAULT_REGION=$AWS_REGION

              aws sts get-caller-identity
              aws ecr get-login-password --region $AWS_REGION \
                | nerdctl login --username AWS --password-stdin $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com

              nerdctl push $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
            '''
          }
        }
      }
    }
  }
}
