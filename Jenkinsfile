pipeline {
  agent any

  environment {
    DEPLOY_HOST = "10.0.0.80"

    APIC_SERVER  = "https://manager.apic12.e1.com"
    APIC_REALM   = "provider/default-idp-2"
    APIC_ORG     = "enterprise1"
    APIC_CATALOG = "skt"

    GATEWAY_SERVICES = "e1-nano-gateway1"

    // Space 안 쓰면 그대로 두면 됨
    USE_SPACE  = "false"
    SPACE_NAME = ""
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Publish Project') {
      steps {
        sshagent(credentials: ['ssh-k8s-worker2']) {
          withCredentials([usernamePassword(
            credentialsId: 'apic-cred',
            usernameVariable: 'APIC_USER',
            passwordVariable: 'APIC_PASS'
          )]) {

            sh '''#!/usr/bin/env bash
set -euo pipefail

# Jenkins env에서 빈 값이 누락되는 경우 대비(핵심)
: "${USE_SPACE:=false}"
: "${SPACE_NAME:=}"

echo "=== Credential sanity check ==="
echo "APIC_USER=${APIC_USER}"
echo "APIC_PASS length=${#APIC_PASS}"

WORK="/tmp/apic_proj_${BUILD_TAG}"
ZIP="${WORK}/project.zip"

ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} "rm -rf '${WORK}' && mkdir -p '${WORK}'"
scp -o StrictHostKeyChecking=no -r apic_cicd root@${DEPLOY_HOST}:"${WORK}/"

ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} \
  env WORK="${WORK}" \
      ZIP="${ZIP}" \
      APIC_SERVER="${APIC_SERVER}" \
      APIC_REALM="${APIC_REALM}" \
      APIC_ORG="${APIC_ORG}" \
      APIC_CATALOG="${APIC_CATALOG}" \
      GATEWAY_SERVICES="${GATEWAY_SERVICES}" \
      USE_SPACE="${USE_SPACE}" \
      SPACE_NAME="${SPACE_NAME}" \
      APIC_USER="${APIC_USER}" \
      APIC_PASS="${APIC_PASS}" \
  bash -s <<'EOS'
set -euo pipefail

: "${USE_SPACE:=false}"
: "${SPACE_NAME:=}"

cd "${WORK}"

command -v zip >/dev/null 2>&1 || { echo "ERROR: zip command not found on deploy host"; exit 10; }

rm -f "${ZIP}"
zip -r "${ZIP}" apic_cicd >/dev/null

apic version
apic login --insecure-skip-tls-verify --server "${APIC_SERVER}" --realm "${APIC_REALM}" -u "${APIC_USER}" -p "${APIC_PASS}"

echo "=== Publishing project zip ==="
if [ "${USE_SPACE}" = "true" ] && [ -n "${SPACE_NAME}" ]; then
  apic projects:publish \
    --insecure-skip-tls-verify \
    --server "${APIC_SERVER}" \
    --org "${APIC_ORG}" \
    --catalog "${APIC_CATALOG}" \
    --scope space \
    --gateway_services "${GATEWAY_SERVICES}" \
    --migrate_subscriptions \
    "${ZIP}"
else
  apic projects:publish \
    --insecure-skip-tls-verify \
    --server "${APIC_SERVER}" \
    --org "${APIC_ORG}" \
    --catalog "${APIC_CATALOG}" \
    --gateway_services "${GATEWAY_SERVICES}" \
    --migrate_subscriptions \
    "${ZIP}"
fi
EOS

ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} "rm -rf '${WORK}'"
'''
          }
        }
      }
    }
  }
}