pipeline {
  agent any
  options { timestamps() }
  environment {
    IMAGE = 'ghcr.io/charan1926/devops-project'
    SHA   = "${env.GIT_COMMIT?.take(7) ?: 'local'}"
    DEV_RELEASE = 'app-dev'
  }
  stages {
    stage('Build Image') {
      steps {
        sh '''cat > Dockerfile <<'DOCKER'
FROM python:3.11-slim
WORKDIR /app
COPY . /app
EXPOSE 8080
CMD ["python3","-m","http.server","8080"]
DOCKER'''
        sh 'docker build -t ${IMAGE}:${SHA} .'
      }
    }
    stage('Login & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'ghcr-creds', usernameVariable: 'USR', passwordVariable: 'PAT')]) {
          sh 'echo $PAT | docker login ghcr.io -u $USR --password-stdin'
          sh 'docker push ${IMAGE}:${SHA}'
        }
      }
    }
    stage('Helm Deploy to DEV') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            helm upgrade --install ${DEV_RELEASE} charts/app \
              -n dev \
              --create-namespace \
              -f env/dev/values.yaml \
              --set image.repository=${IMAGE} \
              --set image.tag=${SHA}

            kubectl rollout status deploy/${DEV_RELEASE}-app -n dev --timeout=120s
            kubectl run smoke --image=busybox:1.36 -n dev --restart=Never -- /bin/sh -c "wget -qO- http://${DEV_RELEASE}-app:8080 | head -n1 || exit 1"
            kubectl delete pod smoke -n dev --ignore-not-found=true
          '''
        }
      }
    }
  }
  post {
    success { echo "✅ Dev deploy OK: ${IMAGE}:${SHA}" }
    failure { echo "❌ Dev deploy failed — check build/push/helm" }
  }
}
