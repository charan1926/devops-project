pipeline {
  agent any
  options {
    timestamps()
    // allow manual override paths to run even if earlier stages are unstable/failed
    skipStagesAfterUnstable(false)
  }
  parameters {
    booleanParam(name: 'FORCE_PROD', defaultValue: false, description: 'Force PROD deployment even if earlier stages failed')
  }

  environment {
    IMAGE         = 'ghcr.io/charan1926/devops-project'
    DEV_RELEASE   = 'app-dev'
    STAGE_RELEASE = 'app-stage'
    PROD_RELEASE  = 'app-prod'
    PROM_CRED_ID  = 'prom-url'
    GHCR_CRED_ID  = 'ghcr-creds'
    KUBE_CRED_ID  = 'kubeconfig'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          // Compute short SHA reliably at runtime
          env.SHA = (env.GIT_COMMIT ?: sh(returnStdout: true, script: "git rev-parse --short=7 HEAD").trim()).take(7)
        }
        echo "Commit SHA: ${env.SHA}"
      }
    }

    stage('Build Image') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
cat > Dockerfile <<'DOCKER'
FROM python:3.11-slim
WORKDIR /app
COPY . /app
EXPOSE 8080
CMD ["python3","-m","http.server","8080"]
DOCKER
docker build -t "$IMAGE:$SHA" .
'''
      }
    }

    stage('Login & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: env.GHCR_CRED_ID, usernameVariable: 'GHCR_USER', passwordVariable: 'GHCR_PAT')]) {
          sh '''#!/usr/bin/env bash
set -euo pipefail
echo "$GHCR_PAT" | docker login ghcr.io -u "$GHCR_USER" --password-stdin
docker push "$IMAGE:$SHA"
'''
        }
      }
    }

    stage('Helm Deploy to DEV (auto)') {
      steps {
        withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
          sh '''#!/usr/bin/env bash
set -euo pipefail

helm upgrade --install "$DEV_RELEASE" charts/app -n dev --create-namespace \
  -f env/dev/values.yaml \
  --set image.repository="$IMAGE" \
  --set image.tag="$SHA"

kubectl rollout status deploy/"$DEV_RELEASE"-app -n dev --timeout=180s

# Run a short-lived smoke pod that should finish quickly
kubectl run smoke -n dev --restart=Never --image=busybox:1.36 -- /bin/sh -c "wget -qO- http://$DEV_RELEASE-app:8080 | head -n1; sleep 2"

timeout_seconds=60; interval=3; elapsed=0
while true; do
  phase=$(kubectl get pod smoke -n dev -o jsonpath='{.status.phase}' 2>/dev/null || echo "")
  echo "dev smoke pod phase: ${phase}"
  if [ "${phase}" = "Succeeded" ]; then
    echo "DEV SMOKE SUCCEEDED"
    break
  fi
  if [ "${phase}" = "Failed" ] || [ "$elapsed" -ge $timeout_seconds ]; then
    echo "DEV SMOKE TIMEOUT/FAILED - collect debug"
    kubectl describe pod smoke -n dev || true
    kubectl logs pod/smoke -n dev --all-containers=true || true
    kubectl get events -n dev --sort-by=.lastTimestamp | tail -n 50 || true
    kubectl delete pod smoke -n dev --ignore-not-found || true
    exit 1
  fi
  sleep $interval
  elapsed=$((elapsed + interval))
done

kubectl logs pod/smoke -n dev > /tmp/dev-smoke.out 2>&1 || true
kubectl delete pod smoke -n dev --ignore-not-found || true

if grep -q "<!DOCTYPE HTML>" /tmp/dev-smoke.out; then
  echo "DEV SMOKE OK"
else
  echo "DEV SMOKE FAILED - output was:"
  sed -n '1,200p' /tmp/dev-smoke.out || true
  exit 1
fi
'''
        }
      }
    }

    stage('Prepare Promotion Metadata') {
      steps {
        script {
          def report = [
            commit: env.GIT_COMMIT ?: env.SHA ?: 'local',
            image : "${env.IMAGE}:${env.SHA}",
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
          def approval = input id: 'PromoteToStage', message: 'Approve promotion to STAGE', parameters: [
            string(name: 'IMAGE_DIGEST', defaultValue: "${IMAGE}:${SHA}", description: 'Image digest (use <repo>:<sha>)'),
            string(name: 'CHANGE_SUMMARY', defaultValue: '', description: 'One-line change summary'),
            string(name: 'APPROVER', defaultValue: "${env.BUILD_USER ?: 'manual'}", description: 'Approver name'),
            string(name: 'TARGET_REPLICAS', defaultValue: '3', description: 'Final replicas for stage')
          ]

          echo "Promotion approved. Inputs: ${approval}"

          // compute IMAGE_TAG in Groovy (safe)
          def imageDigest = approval.IMAGE_DIGEST as String
          def imageTag = imageDigest.contains(':') ? imageDigest.split(':')[-1] : env.SHA

          // Deploy canary (replicaCount=1)
          withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
            sh """#!/usr/bin/env bash
set -euo pipefail
IMAGE_TAG='${imageTag}'
helm upgrade --install "$STAGE_RELEASE" charts/app -n stage --create-namespace \
  -f env/stage/values.yaml \
  --set image.repository="$IMAGE" \
  --set image.tag="$IMAGE_TAG" \
  --set replicaCount=1

kubectl rollout status deploy/"$STAGE_RELEASE"-app -n stage --timeout=180s
"""
          }

          // Stage smoke (robust)
          withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
            sh '''#!/usr/bin/env bash
set -euo pipefail
kubectl run smoke -n stage --restart=Never --image=busybox:1.36 -- /bin/sh -c "wget -qO- http://app-stage-app:8080 | head -n1; sleep 2"

timeout_seconds=60; interval=3; elapsed=0
while true; do
  phase=$(kubectl get pod smoke -n stage -o jsonpath='{.status.phase}' 2>/dev/null || echo "")
  echo "stage smoke pod phase: ${phase}"
  if [ "${phase}" = "Succeeded" ]; then
    echo "STAGE SMOKE SUCCEEDED"
    break
  fi
  if [ "${phase}" = "Failed" ] || [ "$elapsed" -ge $timeout_seconds ]; then
    echo "STAGE SMOKE TIMEOUT/FAILED - collect debug"
    kubectl describe pod smoke -n stage || true
    kubectl logs pod/smoke -n stage --all-containers=true || true
    kubectl get events -n stage --sort-by=.lastTimestamp | tail -n 50 || true
    kubectl delete pod smoke -n stage --ignore-not-found || true
    exit 1
  fi
  sleep $interval
  elapsed=$((elapsed + interval))
done

kubectl logs pod/smoke -n stage > /tmp/stage-smoke.out 2>&1 || true
kubectl delete pod smoke -n stage --ignore-not-found || true

if grep -q "<!DOCTYPE HTML>" /tmp/stage-smoke.out; then
  echo "STAGE SMOKE OK"
else
  echo "STAGE SMOKE FAILED - output was:"
  sed -n '1,200p' /tmp/stage-smoke.out || true
  exit 1
fi
'''
          }

          // Watch metrics from Prometheus for watch window (10 minutes)
          def watchMin = 10
          def breach = false

          withCredentials([string(credentialsId: env.PROM_CRED_ID, variable: 'PROM_URL')]) {
            for (int i = 0; i < watchMin; i++) {
              def errJson = sh(returnStdout: true, script: '''#!/usr/bin/env bash
set -euo pipefail
errQuery='100 * ( sum(rate(http_requests_total{job=~"app.*",status=~"5.."}[5m])) / sum(rate(http_requests_total{job=~"app.*"}[5m])) )'
curl -s -G "$PROM_URL/api/v1/query" --data-urlencode "query=${errQuery}"
''').trim()

              def p95Json = sh(returnStdout: true, script: '''#!/usr/bin/env bash
set -euo pipefail
p95Query='histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job=~"app.*"}[5m])) by (le))'
curl -s -G "$PROM_URL/api/v1/query" --data-urlencode "query=${p95Query}"
''').trim()

              def errVal = 0.0
              def p95Val = 0.0
              try {
                def errParsed = readJSON(text: errJson)
                def p95Parsed = readJSON(text: p95Json)
                if (errParsed.data?.result?.size() > 0) {
                  errVal = errParsed.data.result[0].value[1].toFloat()
                }
                if (p95Parsed.data?.result?.size() > 0) {
                  p95Val = p95Parsed.data.result[0].value[1].toFloat()
                }
              } catch (e) {
                echo "Prometheus read error: ${e}. errJson=${errJson.take(500)} p95Json=${p95Json.take(500)}"
              }

              echo "Prometheus check ${i+1}/${watchMin}: error_rate=${errVal}%, p95=${p95Val}s"

              if (errVal > 2.0 || p95Val > 0.5) {
                breach = true
                echo "SLO breach detected: err=${errVal}, p95=${p95Val}"
                break
              }
              sleep time: 60, unit: 'SECONDS'
            }
          } // end PROM_URL

          if (breach) {
            echo 'Detected breach — rolling back stage canary'
            withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
              sh '''#!/usr/bin/env bash
set -euo pipefail
prev_rev=$(helm history "$STAGE_RELEASE" -n stage --max 2 --output json | jq -r '.[1].revision // .[0].revision' )
if [ -z "$prev_rev" ]; then
  echo "No previous helm revision found, aborting rollback"
  exit 1
fi
helm rollback "$STAGE_RELEASE" "$prev_rev" -n stage
'''
            }
            error "Promotion aborted: SLO breach"
          } else {
            echo 'No breach — scaling to final replicas'
            withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
              sh """#!/usr/bin/env bash
set -euo pipefail
IMAGE_TAG='${imageTag}'
TARGET_REPLICAS='${approval.TARGET_REPLICAS}'
helm upgrade --install "$STAGE_RELEASE" charts/app -n stage \
  -f env/stage/values.yaml \
  --set image.repository="$IMAGE" \
  --set image.tag="$IMAGE_TAG" \
  --set replicaCount=$TARGET_REPLICAS
kubectl rollout status deploy/"$STAGE_RELEASE"-app -n stage --timeout=240s
"""
            }
            echo "Stage promotion completed successfully."
          }
        } // end script
      } // end steps
    } // end Promote to STAGE

    stage('Release to PROD (tag-driven)') {
      when {
        expression {
          // Run if there's a tag OR user explicitly set FORCE_PROD to true
          return (env.GIT_TAG ?: '') as boolean || (params.FORCE_PROD == true)
        }
      }
      steps {
        script {
          // If build already failed/unstable and user didn't set FORCE_PROD, skip to be safe.
          if ((currentBuild.currentResult == 'FAILURE' || currentBuild.currentResult == 'UNSTABLE') && !params.FORCE_PROD) {
            echo "Pipeline is in ${currentBuild.currentResult} state. Skipping PROD unless FORCE_PROD is true."
            error "Skipping PROD due to earlier failures"
          }

          // If forcing despite failures, require an extra confirmation
          if ((currentBuild.currentResult == 'FAILURE' || currentBuild.currentResult == 'UNSTABLE') && params.FORCE_PROD) {
            input message: "Pipeline is ${currentBuild.currentResult}. You have set FORCE_PROD=true. Confirm you want to PROCEED to PROD (this will be recorded)."
          }

          def tag = env.GIT_TAG ?: env.BRANCH_NAME
          echo "Releasing tag: ${tag}"

          input message: "Approve release ${tag} to PROD?", parameters: [string(name:'CHANGELOG', defaultValue:'', description:'Release notes')]

          withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG')]) {
            sh '''#!/usr/bin/env bash
set -euo pipefail
helm upgrade --install "$PROD_RELEASE" charts/app -n prod --create-namespace \
  -f env/prod/values.yaml \
  --set image.repository="$IMAGE" \
  --set image.tag="$SHA" \
  --set replicaCount=3
kubectl rollout status deploy/"$PROD_RELEASE"-app -n prod --timeout=300s
'''
          }

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
