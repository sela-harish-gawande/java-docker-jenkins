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
    AWS_CREDENTIALS = credentials('aws-credentials')

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

    stage('BuildKit daemon (prep)') {
      steps {
        sh '''
          echo "Starting buildkitd (best-effort)"
          mkdir -p /run/buildkit || true
          (buildkitd --root /run/buildkit >/dev/null 2>&1 &) || true
          sleep 2
          nerdctl images || true
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                             credentialsId: 'aws-credentials',
                             secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '''
            set -euxo pipefail
            export AWS_DEFAULT_REGION=$AWS_REGION

            aws sts get-caller-identity >/dev/null
            aws ecr get-login-password --region $AWS_REGION \
              | nerdctl login --username AWS --password-stdin $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com

            echo "Building image $ECR_REPO_NAME:$IMAGE_TAG"
            nerdctl build -t $ECR_REPO_NAME:$IMAGE_TAG .
            nerdctl tag  $ECR_REPO_NAME:$IMAGE_TAG $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
          '''
        }
      }
    }

    stage('Push to ECR') {
      steps {
        withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                             credentialsId: 'aws-credentials',
                             secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '''
            set -euxo pipefail
            export AWS_DEFAULT_REGION=$AWS_REGION

            aws sts get-caller-identity >/dev/null
            aws ecr get-login-password --region $AWS_REGION \
              | nerdctl login --username AWS --password-stdin $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com

            nerdctl push $Account_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
          '''
        }
      }
    }

    // --------- Compile code for Sonar (Java) ---------
    stage('Compile for Sonar (Java)') {
      steps {
        sh '''
          set -euxo pipefail
          SONAR_BINARIES=""

          if [ -f "./mvnw" ]; then
            chmod +x ./mvnw
            ./mvnw -B -DskipTests=false -Dmaven.compiler.release=17 clean verify
            SONAR_BINARIES="**/target/classes"

          elif [ -f "pom.xml" ]; then
            mvn -v >/dev/null 2>&1 || true
            mvn -B -DskipTests=false -Dmaven.compiler.release=17 clean verify
            SONAR_BINARIES="**/target/classes"

          elif [ -f "./gradlew" ]; then
            chmod +x ./gradlew
            ./gradlew clean build -Dorg.gradle.jvmargs="--release 17"
            SONAR_BINARIES="**/build/classes/java/main"

          elif [ -f "build.gradle" ]; then
            gradle -v >/dev/null 2>&1 || true
            gradle clean build -Dorg.gradle.jvmargs="--release 17"
            SONAR_BINARIES="**/build/classes/java/main"
          fi

          echo "$SONAR_BINARIES" > .sonar_binaries_path
        '''
      }
    }

    // ----------------- SonarQube Analysis -----------------
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          withEnv(["PATH+SONAR=${tool SONAR_SCANNER}/bin"]) {
            sh '''
              set -euxo pipefail

              SONAR_BINARIES="$(cat .sonar_binaries_path || true)"
              SONAR_BIN_FLAG=""
              JAVA_EXCLUDE_FLAG=""

              if [ -n "$SONAR_BINARIES" ]; then
                SONAR_BIN_FLAG="-Dsonar.java.binaries=${SONAR_BINARIES}"
              else
                # Fallback: no compiler available → avoid Java sensor failure
                JAVA_EXCLUDE_FLAG="-Dsonar.exclusions=**/*.java,**/node_modules/**,**/dist/**,**/target/**,**/.git/**"
              fi

              sonar-scanner \
                -Dsonar.projectKey=plumhq-jenkins-sample \
                -Dsonar.projectName="PlumHQ Jenkins Sample" \
                -Dsonar.sources=. \
                ${SONAR_BIN_FLAG} \
                ${JAVA_EXCLUDE_FLAG} \
                -Dsonar.host.url=$SONAR_HOST_URL \
                -Dsonar.token=$SONAR_AUTH_TOKEN

              # Add coverage if you produce reports:
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
  }
}
