pipeline {
  agent any

  tools {
    maven 'maven'   // Must match Manage Jenkins > Global Tool Configuration
  }

  environment {
    // ==== EDIT ONLY IF YOUR NAMES DIFFER ====
    GIT_BRANCH   = 'main'
    GIT_URL      = 'https://github.com/luckysuie/luckyspringpetclinic.git'

    // SonarCloud
    SONAR_ORG     = 'vnitesh26'          // org key (lowercase)
    SONAR_PROJECT = 'jenkinsproject'     // project key
    SONAR_HOST    = 'https://sonarcloud.io'

    // Container registry & image
    ACR_NAME   = 'luckyregistry1n'
    ACR_SERVER = "${ACR_NAME}.azurecr.io"
    IMAGE_NAME = 'my-app-image'
    IMAGE_TAG  = 'latest'                // or BUILD_NUMBER / short SHA

    // AKS target
    AKS_RG      = 'demo11'
    AKS_CLUSTER = 'lucky-aks-cluster11'
  }

  stages {
    stage('Checkout From Git') {
      steps {
        git url: "${GIT_URL}", branch: "${GIT_BRANCH}"
      }
    }

    stage('Maven Validate') {
      steps { sh 'mvn -B -U validate' }
    }

    stage('Maven Compile') {
      steps { sh 'mvn -B compile' }
    }

    stage('Maven Test') {
      steps { sh 'mvn -B test' }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
        }
      }
    }

    stage('Maven Package') {
      steps { sh 'mvn -B package -DskipTests=false' }
      post {
        success {
          archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true, allowEmptyArchive: true
        }
      }
    }

    stage('SonarCloud Analysis') {
      steps {
        withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
          sh '''
            mvn -B sonar:sonar \
              -Dsonar.organization=${SONAR_ORG} \
              -Dsonar.projectKey=${SONAR_PROJECT} \
              -Dsonar.host.url=${SONAR_HOST} \
              -Dsonar.token=$SONAR_TOKEN
          '''
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
          docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${ACR_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}
          docker image ls ${ACR_SERVER}/${IMAGE_NAME}
        '''
      }
    }

    stage('Trivy Scan (HIGH/CRITICAL)') {
      steps {
        // Scans the locally-tagged image (no push required yet)
        sh '''
          trivy --version || true
          trivy image --severity HIGH,CRITICAL \
            --format table \
            --output trivy-report.txt \
            ${ACR_SERVER}/${IMAGE_NAME}:${IMAGE_TAG} || true
          cat trivy-report.txt || true
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
        }
      }
    }

    // ======= REPLACED STAGE: uses ACR admin credentials (no az needed) =======
    stage('Login to ACR & Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'acr-credentials', usernameVariable: 'ACR_USER', passwordVariable: 'ACR_PASS')]) {
          sh '''
            set -e
            echo "$ACR_PASS" | docker login ${ACR_SERVER} -u "$ACR_USER" --password-stdin
            docker push ${ACR_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}
            docker logout ${ACR_SERVER} || true
          '''
        }
      }
    }
    // ========================================================================

    stage('Deploy to Kubernetes') {
      steps {
        // Uses your existing Service Principal for AKS auth
        withCredentials([
          usernamePassword(credentialsId: 'azure-sp', usernameVariable: 'AZURE_APP_ID', passwordVariable: 'AZURE_PASSWORD'),
          string(credentialsId: 'azure-tenant', variable: 'AZURE_TENANT')
        ]) {
          sh '''
            set -e
            az login --service-principal -u "$AZURE_APP_ID" -p "$AZURE_PASSWORD" --tenant "$AZURE_TENANT" > /dev/null
            az aks get-credentials --resource-group ${AKS_RG} --name ${AKS_CLUSTER} --overwrite-existing

            # Apply all manifests in k8s/ (make sure they exist; you already have k8s/db.yml)
            kubectl apply -f k8s/ || true

            # Ensure the running deployment uses the just-built image
            kubectl set image deployment/webapp webapp=${ACR_SERVER}/${IMAGE_NAME}:${IMAGE_TAG} --record -n default || true

            kubectl rollout status deployment/webapp -n default || true
            kubectl get svc,deploy,pods -o wide
          '''
        }
      }
    }
  }

  post {
    success {
      echo "SUCCESS ✅  Image: ${ACR_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}"
      echo "SonarCloud: ${SONAR_HOST}/project/overview?id=${SONAR_PROJECT}"
    }
    failure {
      echo "FAILED ❌  Check stage logs above."
    }
  }
}
