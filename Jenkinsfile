pipeline {
  agent any
  triggers { githubPush() }
  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }
  environment {
    AWS_DEFAULT_REGION = 'ap-south-1'
    EKS_CLUSTER = 'travel-easy-cluster'
    K8S_NAMESPACE = 'travel-easy'
    DOCKERHUB_USERNAME = 'REPLACE_DOCKERHUB_USERNAME'
    IMAGE_NAME = 'travel-easy'
    IMAGE = "${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}"
  }
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Install and Test') {
      steps {
        sh '''
          python3 -m venv .venv
          . .venv/bin/activate
          pip install --upgrade pip
          pip install -r requirements.txt
          python manage.py check
        '''
      }
    }
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh 'sonar-scanner -Dsonar.projectKey=travel-easy -Dsonar.sources=app,project'
        }
      }
    }
    stage('Trivy Filesystem Scan') {
      steps { sh 'trivy fs --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed .' }
    }
    stage('Docker Build') {
      steps { sh 'docker build -t "$IMAGE" .' }
    }
    stage('Trivy Image Scan') {
      steps { sh 'trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed "$IMAGE"' }
    }
    stage('Docker Hub Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_TOKEN')]) {
          sh '''
            echo "$DH_TOKEN" | docker login --username "$DH_USER" --password-stdin
            docker push "$IMAGE"
            docker logout
          '''
        }
      }
    }
    stage('Configure EKS') {
      steps {
        sh '''
          aws eks update-kubeconfig --region "$AWS_DEFAULT_REGION" --name "$EKS_CLUSTER"
          kubectl get nodes
        '''
      }
    }
    stage('Deploy to EKS') {
      steps {
        sh '''
          kubectl create namespace "$K8S_NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
          kubectl apply -f k8s/deployment.yaml
          kubectl apply -f k8s/service.yaml
          kubectl set image deployment/travel-easy travel-easy="$IMAGE" -n "$K8S_NAMESPACE"
          kubectl rollout status deployment/travel-easy -n "$K8S_NAMESPACE" --timeout=180s
        '''
      }
    }
  }
  post {
    success {
      emailext(to: 'REPLACE_WITH_EMAIL', subject: "SUCCESS: ${JOB_NAME} #${BUILD_NUMBER}", body: "Travel Eazy pipeline succeeded. Image: ${IMAGE}")
    }
    failure {
      emailext(to: 'REPLACE_WITH_EMAIL', subject: "FAILED: ${JOB_NAME} #${BUILD_NUMBER}", body: "Travel Eazy pipeline failed. Check Jenkins console output.")
    }
    always {
      sh 'docker image prune -f || true'
      cleanWs()
    }
  }
}
