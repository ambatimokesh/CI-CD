pipeline {

  agent any

  triggers {
    githubPush()
  }

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  environment {

    AWS_DEFAULT_REGION = 'ap-south-1'
    AWS_REGION         = 'ap-south-1'

    EKS_CLUSTER        = 'travel-easy-cluster'
    K8S_NAMESPACE      = 'travel-easy'

    DOCKERHUB_USERNAME = 'mokeshambati'
    IMAGE_NAME         = 'travel-easy'
    IMAGE_TAG          = "${BUILD_NUMBER}"
    DOCKER_IMAGE       = "mokeshambati/travel-easy:${BUILD_NUMBER}"

    KUBECONFIG         = "${WORKSPACE}/jenkins-kubeconfig"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Python Setup') {
      steps {
        sh '''
          set -e

          echo "===== PYTHON VERSION ====="
          python3 --version

          echo "===== CREATE VIRTUAL ENVIRONMENT ====="
          python3 -m venv .venv

          . .venv/bin/activate

          echo "===== UPGRADE PIP ====="
          python -m pip install --upgrade pip

          echo "===== INSTALL REQUIREMENTS ====="
          pip install -r requirements.txt

          echo "===== INSTALLED PACKAGES ====="
          pip list
        '''
      }
    }

    stage('Django Checks') {
      steps {
        sh '''
          set -e

          . .venv/bin/activate

          echo "===== DJANGO CHECK ====="
          python manage.py check

          echo "===== DJANGO TEST ====="
          python manage.py test
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        script {

          def scannerHome = tool 'SonarScanner'

          withSonarQubeEnv('SonarQube') {

            sh """
              ${scannerHome}/bin/sonar-scanner \
                -Dsonar.projectKey=travel-easy \
                -Dsonar.projectName="Travel Eazy" \
                -Dsonar.sources=app,project \
                -Dsonar.python.version=3.11 \
                -Dsonar.exclusions="**/migrations/**,**/__pycache__/**,**/static/**,**/templates/**"
            """
          }
        }
      }
    }

    stage('Trivy Filesystem Scan') {
      steps {
        sh '''
          echo "========================================"
          echo "TRIVY FILESYSTEM SECURITY SCAN"
          echo "========================================"

          trivy fs \
            --scanners vuln,secret \
            --severity HIGH,CRITICAL \
            --exit-code 0 \
            --ignore-unfixed \
            .
        '''
      }
    }

    stage('Docker Build') {
      steps {
        sh '''
          set -e

          echo "========================================"
          echo "DOCKER BUILD"
          echo "========================================"

          docker build \
            -t ${DOCKER_IMAGE} \
            .

          echo "===== DOCKER IMAGE ====="

          docker images ${DOCKERHUB_USERNAME}/${IMAGE_NAME}
        '''
      }
    }

    stage('Trivy Image Scan') {
      steps {
        sh '''
          echo "========================================"
          echo "TRIVY DOCKER IMAGE SECURITY SCAN"
          echo "========================================"

          trivy image \
            --severity HIGH,CRITICAL \
            --exit-code 0 \
            --ignore-unfixed \
            ${DOCKER_IMAGE}
        '''
      }
    }

    stage('Docker Hub Push') {
      steps {

        withCredentials([
          usernamePassword(
            credentialsId: 'dockerhub-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASSWORD'
          )
        ]) {

          sh '''
            set -e

            echo "========================================"
            echo "DOCKER HUB LOGIN"
            echo "========================================"

            echo "$DOCKER_PASSWORD" | docker login \
              --username "$DOCKER_USER" \
              --password-stdin

            echo "===== PUSH IMAGE ====="

            docker push ${DOCKER_IMAGE}

            echo "===== LOGOUT ====="

            docker logout
          '''
        }
      }
    }

    stage('Configure EKS') {
      steps {
        sh '''
          set -e

          echo "========================================"
          echo "AWS IDENTITY"
          echo "========================================"

          aws sts get-caller-identity

          echo "========================================"
          echo "CONFIGURE EKS"
          echo "========================================"

          aws eks update-kubeconfig \
            --region ${AWS_DEFAULT_REGION} \
            --name ${EKS_CLUSTER} \
            --kubeconfig ${KUBECONFIG}

          echo "===== EKS NODES ====="

          kubectl --kubeconfig ${KUBECONFIG} get nodes
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
          set -e

          echo "========================================"
          echo "CREATE / VERIFY NAMESPACE"
          echo "========================================"

          kubectl --kubeconfig ${KUBECONFIG} \
            create namespace ${K8S_NAMESPACE} \
            --dry-run=client \
            -o yaml | \
            kubectl --kubeconfig ${KUBECONFIG} apply -f -

          echo "========================================"
          echo "APPLY DEPLOYMENT"
          echo "========================================"

          kubectl --kubeconfig ${KUBECONFIG} \
            apply -f k8s/deployment.yaml

          echo "========================================"
          echo "APPLY SERVICE"
          echo "========================================"

          kubectl --kubeconfig ${KUBECONFIG} \
            apply -f k8s/service.yaml

          echo "========================================"
          echo "UPDATE APPLICATION IMAGE"
          echo "========================================"

          kubectl --kubeconfig ${KUBECONFIG} \
            -n ${K8S_NAMESPACE} \
            set image deployment/travel-easy \
            travel-easy=${DOCKER_IMAGE}

          echo "========================================"
          echo "WAIT FOR ROLLOUT"
          echo "========================================"

          kubectl --kubeconfig ${KUBECONFIG} \
            -n ${K8S_NAMESPACE} \
            rollout status deployment/travel-easy \
            --timeout=5m
        '''
      }
    }

    stage('Verify Deployment') {
      steps {
        sh '''
          set -e

          echo "========================================"
          echo "KUBERNETES DEPLOYMENT"
          echo "========================================"

          kubectl --kubeconfig ${KUBECONFIG} \
            get deployment \
            -n ${K8S_NAMESPACE}

          echo "========================================"
          echo "KUBERNETES PODS"
          echo "========================================"

          kubectl --kubeconfig ${KUBECONFIG} \
            get pods \
            -n ${K8S_NAMESPACE} \
            -o wide

          echo "========================================"
          echo "KUBERNETES SERVICE"
          echo "========================================"

          kubectl --kubeconfig ${KUBECONFIG} \
            get svc \
            -n ${K8S_NAMESPACE}

          echo "========================================"
          echo "DEPLOYMENT IMAGE"
          echo "========================================"

          kubectl --kubeconfig ${KUBECONFIG} \
            get deployment travel-easy \
            -n ${K8S_NAMESPACE} \
            -o jsonpath='{.spec.template.spec.containers[0].image}'

          echo
        '''
      }
    }
  }

  post {

    success {
      echo """
      ========================================
      TRAVEL EAZY CI/CD PIPELINE SUCCESSFUL
      ========================================

      Docker Image : ${DOCKER_IMAGE}
      EKS Cluster   : ${EKS_CLUSTER}
      Namespace     : ${K8S_NAMESPACE}

      ========================================
      """
    }

    failure {
      echo """
      ========================================
      TRAVEL EAZY CI/CD PIPELINE FAILED
      ========================================

      Check the failed stage in Jenkins Console Output.

      ========================================
      """
    }

    always {
      sh '''
        rm -f "${KUBECONFIG}" || true
        docker image prune -f || true
      '''

      cleanWs()
    }
  }
}