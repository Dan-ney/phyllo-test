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
      steps {   // ✅ ADDED THIS
        container('kaniko') {
          withCredentials([file(credentialsId: 'gcp-sa', variable: 'GCP_KEY')]) {
            sh '''
              echo "⚙️ Building and pushing image with Kaniko..."
              export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY
              cat $GOOGLE_APPLICATION_CREDENTIALS | jq '.' > /dev/null || echo "✅ SA key detected"
              /kaniko/executor \
                --context $PWD \
                --dockerfile $PWD/Dockerfile \
                --destination gcr.io/upheld-pursuit-477207-s4/phyllo-test:${BUILD_NUMBER} \
                --cleanup \
                --single-snapshot \
                --verbosity info
            '''
          }
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
