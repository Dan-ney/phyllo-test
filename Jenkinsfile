pipeline {
  agent any
  environment {
    PROJECT_ID = 'clear-booking-470907-q6'
    REGION = 'us-central1'
    REPO = 'phyllo-test'
    IMAGE = 'demo-app'
    CLUSTER = 'demo-cluster'
    ZONE = 'us-central1-c'
  }
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Build image') {
      steps {
        sh """
        gcloud auth activate-service-account --key-file=/var/lib/jenkins/gcloud-key.json
        gcloud auth configure-docker ${REGION}-docker.pkg.dev -q
        docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE}:${BUILD_NUMBER} .
        """
      }
    }
    stage('Push image') {
      steps {
        sh "docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE}:${BUILD_NUMBER}"
      }
    }
    stage('Deploy to GKE') {
      steps {
        sh """
        gcloud container clusters get-credentials ${CLUSTER} --zone ${ZONE} --project ${PROJECT_ID}
        kubectl set image deployment/demo-app demo=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE}:${BUILD_NUMBER} --record
        kubectl rollout status deployment/demo-app
        """
      }
    }
  }
}
