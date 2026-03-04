pipeline {
  agent any

  environment {
    DEPLOY_HOST = "10.0.0.80"

    APIC_SERVER  = "https://manager.apic12.e1.com"
    APIC_REALM   = "provider/default-idp-2"
    APIC_ORG     = "enterprise1"
    APIC_CATALOG = "skt"

    // Studio에서 쓰던 gateway_services 값으로 설정 (필요 시 콤마로 여러 개)
    GATEWAY_SERVICES = "e1-api-gateway"

    // Space를 쓰면 true로 바꾸고 SPACE_NAME 채우기
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

echo "=== Credential sanity check ==="
echo "APIC_USER=${APIC_USER}"
echo "APIC_PASS length=${#APIC_PASS}"
test -n "${APIC_USER}" || { echo "ERROR: APIC_USER empty"; exit 2; }
test -n "${APIC_PASS}" || { echo "ERROR: APIC_PASS empty"; exit 2; }

WORK="/tmp/apic_proj_${BUILD_TAG}"
ZIP="${WORK}/project.zip"

# 원격 작업 폴더 준비 + 소스 전송
ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} "rm -rf '${WORK}' && mkdir -p '${WORK}'"
scp -o StrictHostKeyChecking=no -r apic_cicd root@${DEPLOY_HOST}:"${WORK}/"

# 원격에서 zip 생성 + projects:publish 실행
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

cd "${WORK}"

# zip 유틸 확인(없으면 설치 필요)
command -v zip >/dev/null 2>&1 || { echo "ERROR: zip command not found on deploy host"; exit 10; }

# 프로젝트 폴더를 zip으로 묶기 (폴더 자체 포함)
rm -f "${ZIP}"
zip -r "${ZIP}" apic_cicd >/dev/null

apic version

# TLS 문제 회피 옵션 포함
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

# 정리
ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} "rm -rf '${WORK}'"
'''
          }
        }
      }
    }
  }
}