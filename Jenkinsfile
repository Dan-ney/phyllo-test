pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent
spec:
  containers:
  - name: kubectl
    image: gcr.io/cloud-builders/kubectl
    command:
      - /bin/sh
      - -c
      - cat
    tty: true

  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    command:
      - /busybox/sh
      - -c
      - cat
    tty: true
    volumeMounts:
      - name: gcp-key
        mountPath: /kaniko/.docker
  volumes:
  - name: gcp-key
    projected:
      sources:
      - secret:
          name: gcp-sa
"""
    }
  }

  environment {
    PROJECT_ID = "upheld-pursuit-477207-s4"
    DOCKER_REPO = "gcr.io/${PROJECT_ID}/phyllo-test"
    IMAGE_TAG = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('Authenticate to GCP') {
      steps {
        container('kubectl') {
          withCredentials([file(credentialsId: 'gcp-sa', variable: 'GCP_KEY')]) {
            sh '''
              echo "🔐 Authenticating to GCP..."
              gcloud auth activate-service-account --key-file=$GCP_KEY
              gcloud config set project $PROJECT_ID
              gcloud auth configure-docker gcr.io --quiet
            '''
          }
        }
      }
    }

    stage('Build & Push with Kaniko') {
      steps {
        container('kaniko') {
          sh '''
            echo "⚙️ Building and pushing image with Kaniko..."
            /kaniko/executor \
              --context `pwd` \
              --dockerfile `pwd`/Dockerfile \
              --destination $DOCKER_REPO:$IMAGE_TAG \
              --cleanup
          '''
        }
      }
    }

    stage('Update Helm Values') {
      steps {
        container('kubectl') {
          sh '''
            echo "📝 Updating Helm values.yaml with new image details..."
            sed -i "s|repository:.*|repository: $DOCKER_REPO|" helm/values.yaml
            sed -i "s|tag:.*|tag: \\"$IMAGE_TAG\\"|" helm/values.yaml

            git config user.email "ci@enhub.ai"
            git config user.name "jenkins"
            git add helm/values.yaml
            git commit -m "Update image to $DOCKER_REPO:$IMAGE_TAG" || echo "No changes to commit"
          '''
        }
      }
      post {
        success {
          withCredentials([string(credentialsId: 'github-token', variable: 'GIT_TOKEN')]) {
            sh '''
              echo "🚀 Pushing Helm update to GitHub..."
              git remote set-url origin https://$GIT_TOKEN@github.com/Dan-ney/phyllo-test.git
              git push origin main
            '''
          }
        }
      }
    }
  }
}
