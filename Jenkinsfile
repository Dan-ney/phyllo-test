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
  serviceAccountName: jenkins
  containers:
  - name: kubectl
    image: gcr.io/google.com/cloudsdktool/cloud-sdk:slim
    command:
      - cat
    tty: true

  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
      - cat
    tty: true
    volumeMounts:
      - name: gcp-key
        mountPath: /kaniko/.docker

  volumes:
  - name: gcp-key
    secret:
      secretName: gcp-sa
"""
    }
  }

  environment {
    PROJECT_ID = "upheld-pursuit-477207-s4"
    IMAGE_NAME = "phyllo-test"
    DOCKER_REPO = "gcr.io/${PROJECT_ID}/${IMAGE_NAME}"
    IMAGE_TAG = "${env.BUILD_NUMBER}"
    HELM_PATH = "phyllo-test/main/helm/values.yaml"
  }

  stages {

    stage('Checkout Repository') {
      steps {
        container('kubectl') {
          echo "📦 Checking out source code..."
          checkout scm
        }
      }
    }

    stage('Authenticate to GCP') {
      steps {
        container('kubectl') {
          withCredentials([file(credentialsId: 'gcp-sa', variable: 'GCP_KEY')]) {
            sh '''
              echo "🔐 Authenticating to GCP..."
              gcloud auth activate-service-account --key-file=$GCP_KEY
              gcloud config set project $PROJECT_ID
              gcloud auth configure-docker gcr.io --quiet
              echo "✅ Authentication successful."
            '''
          }
        }
      }
    }

    stage('Build & Push Image with Kaniko') {
      steps {
        container('kaniko') {
          withCredentials([file(credentialsId: 'gcp-sa', variable: 'GCP_KEY')]) {
            sh '''
              echo "⚙️ Building and pushing image with Kaniko..."
              export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY

              /kaniko/executor \
                --context $PWD \
                --dockerfile $PWD/Dockerfile \
                --destination gcr.io/$PROJECT_ID/$IMAGE_NAME:$BUILD_NUMBER \
                --cleanup \
                --verbosity info

              echo "✅ Image pushed successfully: gcr.io/$PROJECT_ID/$IMAGE_NAME:$BUILD_NUMBER"
            '''
          }
        }
      }
    }

    stage('Update Helm values.yaml') {
      steps {
        container('kubectl') {
          withCredentials([string(credentialsId: 'github-token', variable: 'GIT_TOKEN')]) {
            sh '''
              echo "📝 Updating Helm values.yaml..."
              
              if [ ! -f helm/values.yaml ]; then
                echo "❌ ERROR: helm/values.yaml not found!"
                exit 1
              fi

              sed -i "s|repository:.*|repository: ${DOCKER_REPO}|" helm/values.yaml
              sed -i "s|tag:.*|tag: \\"${IMAGE_TAG}\\"|" helm/values.yaml

              git config --global --add safe.directory $(pwd)
              git config user.email "jenkins@enhub.ai"
              git config user.name "Jenkins CI"

              git fetch origin main
              git checkout main || git checkout -b main
              git pull origin main --rebase

              git add helm/values.yaml
              git commit -m "Update image to ${DOCKER_REPO}:${IMAGE_TAG}" || echo "⚠️ No changes to commit"

              echo "🚀 Pushing changes to GitHub..."
              git remote set-url origin https://${GIT_TOKEN}@github.com/Dan-ney/phyllo-test.git
              git push origin main || echo "⚠️ Nothing new to push"

              echo "✅ Helm values.yaml updated and pushed to GitHub."
            '''
          }
        }
      }
    }
  }
}
