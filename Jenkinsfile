// Jenkinsfile — Dev -> Stage promotion -> (optional) Prod
pipeline {
  agent any
  options { timestamps() }
  environment {
    IMAGE        = 'ghcr.io/charan1926/devops-project'
    SHA          = "${env.GIT_COMMIT?.take(7) ?: 'local'}"
    DEV_RELEASE  = 'app-dev'
    STAGE_RELEASE= 'app-stage'
    PROD_RELEASE = 'app-prod'
    PROM_CRED_ID = 'prom-url'
    GHCR_CRED_ID = 'ghcr-creds'
    KUBE_CRED_ID = 'kubeconfig'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        echo "Commit SHA: ${env.GIT_COMMIT}"
      }
    }

    stage('Build Image') {
      steps {
        // create a real python-based Dockerfile for the demo server
        sh '''cat > Dockerfile <<'DOCKER'
FROM python:3.11-slim
WORKDIR /app
COPY . /app
EXPOSE 8080
CMD ["python3","-m","http.server","8080"]
DOCKER
'''
        sh 'docker build -t ${IMAGE}:${SHA} .'
      }
    }

    stage('Login & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: env.GHCR_CRED_ID, usernameVariable: 'GHCR_USER', passwordVariable: 'GHCR_PAT')]) {
          // avoid Groovy interpolation warnings by using single-quoted scripts
          sh 'echo $GHCR_PAT | docker login ghcr.io -u $GHCR_USER --password-stdin'
          sh 'docker push ${IMAGE}:${SHA}'
        }
      }
    }

    stage('Helm Deploy to DEV (auto)') {
      steps {
        withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
          sh '''
            set -e
            helm upgrade --install ${DEV_RELEASE} charts/app -n dev --create-namespace \
              -f env/dev/values.yaml \
              --set image.repository=${IMAGE} \
              --set image.tag=${SHA}

            # wait for new rollout
            kubectl rollout status deploy/${DEV_RELEASE}-app -n dev --timeout=180s

            # simple in-cluster smoke test
            kubectl run -n dev --rm -i --tty smoke --image=busybox:1.36 --restart=Never -- /bin/sh -c "wget -qO- http://${DEV_RELEASE}-app:8080 | head -n1"
          '''
        }
      }
    }

    stage('Prepare Promotion Metadata') {
      steps {
        script {
          // write a small build-report for auditing
          def report = [
            commit: env.GIT_COMMIT ?: 'local',
            image: "${env.IMAGE}:${env.SHA}",
            timestamp: new Date().toString()
          ]
          writeJSON file: 'build-report.json', json: report
          archiveArtifacts artifacts: 'build-report.json', fingerprint: true
        }
      }
    }

    stage('Promote to STAGE (manual)') {
      steps {
        script {
          // This blocks and shows input form in Jenkins UI
          def approval = input id: 'PromoteToStage', message: 'Approve promotion to STAGE', parameters: [
            string(name: 'IMAGE_DIGEST', defaultValue: "${IMAGE}:${SHA}", description: 'Image digest (use <repo>:<sha>)'),
            string(name: 'CHANGE_SUMMARY', defaultValue: '', description: 'One-line change summary'),
            string(name: 'APPROVER', defaultValue: "${env.BUILD_USER ?: 'manual'}", description: 'Approver name'),
            string(name: 'TARGET_REPLICAS', defaultValue: '3', description: 'Final replicas for stage')
          ]

          echo "Promotion approved. Inputs: ${approval}"

          // Deploy canary (replicaCount = 1)
          withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
            sh """
              set -e
              IMAGE_TAG=\$(echo "${approval.IMAGE_DIGEST}" | awk -F: '{print \$2}')
              helm upgrade --install ${STAGE_RELEASE} charts/app -n stage --create-namespace \
                -f env/stage/values.yaml \
                --set image.repository=${IMAGE} \
                --set image.tag=\$IMAGE_TAG \
                --set replicaCount=1

              kubectl rollout status deploy/${STAGE_RELEASE}-app -n stage --timeout=180s
            """
          }

          // Smoke test
          withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
            sh 'kubectl run -n stage --rm -i --tty smoke --image=busybox:1.36 --restart=Never -- /bin/sh -c "wget -qO- http://app-stage-app:8080 | head -n1"'
          }

          // Watch metrics from Prometheus for watch window (10 minutes)
          def promUrl = credentials(env.PROM_CRED_ID)
          def watchMin = 10
          def breach = false
          for (int i = 0; i < watchMin; i++) {
            // PromQL queries (URL-encoded below)
            def errQuery = URLEncoder.encode('100 * ( sum(rate(http_requests_total{job=~"app.*",status=~"5.."}[5m])) / sum(rate(http_requests_total{job=~"app.*"}[5m])) )', 'UTF-8')
            def p95Query = URLEncoder.encode('histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job=~"app.*"}[5m])) by (le))', 'UTF-8')

            def errJson = sh(returnStdout: true, script: "curl -s '${promUrl}/api/v1/query?query=${errQuery}'") .trim()
            def p95Json = sh(returnStdout: true, script: "curl -s '${promUrl}/api/v1/query?query=${p95Query}'") .trim()

            def errVal = 0.0
            def p95Val = 0.0
            try {
              errVal = readJSON(text: errJson).data.result.size() ? readJSON(text: errJson).data.result[0].value[1].toFloat() : 0.0
              p95Val = readJSON(text: p95Json).data.result.size() ? readJSON(text: p95Json).data.result[0].value[1].toFloat() : 0.0
            } catch (e) {
              echo "Prometheus read error: ${e}. errJson=${errJson} p95Json=${p95Json}"
            }

            echo "Prometheus check ${i+1}/${watchMin}: error_rate=${errVal}%, p95=${p95Val}s"

            if (errVal > 2.0 || p95Val > 0.5) {
              breach = true
              echo "SLO breach detected: err=${errVal}, p95=${p95Val}"
              break
            }
            sleep time: 60, unit: 'SECONDS'
          }

          if (breach) {
            echo 'Detected breach — rolling back stage canary'
            withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
              // rollback to previous helm revision
              sh """
                set -e
                helm rollback ${STAGE_RELEASE} \$(helm history ${STAGE_RELEASE} -n stage --max 2 --output json | jq -r '.[0].revision')
              """
            }
            error "Promotion aborted: SLO breach"
          } else {
            echo 'No breach — scaling to final replicas'
            withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
              sh """
                set -e
                IMAGE_TAG=\$(echo "${approval.IMAGE_DIGEST}" | awk -F: '{print \$2}')
                helm upgrade --install ${STAGE_RELEASE} charts/app -n stage \
                  -f env/stage/values.yaml \
                  --set image.repository=${IMAGE} \
                  --set image.tag=\$IMAGE_TAG \
                  --set replicaCount=${approval.TARGET_REPLICAS}
                kubectl rollout status deploy/${STAGE_RELEASE}-app -n stage --timeout=240s
              """
            }
            echo "Stage promotion completed successfully."
          }
        } // end script
      } // end steps
    } // end Stage promotion

    // Optional: Prod release stage; runs only when this job is built for a tag
    stage('Release to PROD (tag-driven)') {
      when {
        expression { return env.GIT_TAG ?: false } // runs when Jenkins builds a tag (may vary by setup)
      }
      steps {
        script {
          // Validate tag / produce release notes
          def tag = env.GIT_TAG ?: env.BRANCH_NAME
          echo "Releasing tag: ${tag}"

          def imageToUse = "${IMAGE}:${SHA}" // adapt if you store digest per build

          // require an approval (simple input)
          input message: "Approve release ${tag} to PROD?", parameters: [string(name:'CHANGELOG', defaultValue:'', description:'Release notes')]

          withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
            sh """
              set -e
              helm upgrade --install ${PROD_RELEASE} charts/app -n prod --create-namespace \
                -f env/prod/values.yaml \
                --set image.repository=${IMAGE} \
                --set image.tag=${SHA} \
                --set replicaCount=3
              kubectl rollout status deploy/${PROD_RELEASE}-app -n prod --timeout=300s
            """
          }

          // Post-release: you can call a script to snapshot dashboards or create GitHub Release
          echo "Prod release done for ${tag}"
        }
      }
    }

  } // end stages

  post {
    success {
      echo "Pipeline SUCCESS: ${IMAGE}:${SHA}"
    }
    failure {
      echo "Pipeline FAILED: check logs and fix"
    }
    always {
      archiveArtifacts artifacts: 'build-report.json', allowEmptyArchive: true
    }
  }
}

