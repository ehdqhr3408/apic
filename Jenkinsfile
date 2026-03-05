pipeline {
  agent any

  environment {
    DEPLOY_HOST = "10.0.0.80"

    APIC_SERVER  = "https://manager.apic12.e1.com"
    APIC_REALM   = "provider/default-idp-2"
    APIC_ORG     = "enterprise1"
    APIC_CATALOG = "skt"

    GATEWAY_SERVICES = "e1-api-gateway"

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

: "${USE_SPACE:=false}"
: "${SPACE_NAME:=}"

echo "=== Credential sanity check ==="
echo "APIC_USER=${APIC_USER}"
echo "APIC_PASS length=${#APIC_PASS}"

WORK="/tmp/apic_proj_${BUILD_TAG}"
ARCHIVE="${WORK}/build.zip"

# 원격 작업 폴더 준비 + 소스 전송
ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} "rm -rf '${WORK}' && mkdir -p '${WORK}'"
scp -o StrictHostKeyChecking=no -r apic_cicd root@${DEPLOY_HOST}:"${WORK}/"
scp -o StrictHostKeyChecking=no .apistudio-projects root@${DEPLOY_HOST}:"${WORK}/"

# 원격에서 apic build -> projects:publish
ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} \
  env WORK="${WORK}" \
      ARCHIVE="${ARCHIVE}" \
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

echo "=== Remote environment ==="
echo "WORK=${WORK}"
echo "ARCHIVE=${ARCHIVE}"
echo "node=$(node -v 2>/dev/null || echo NONE)"
apic version

# 로그인(TLS 스킵)
apic login --insecure-skip-tls-verify --server "${APIC_SERVER}" --realm "${APIC_REALM}" -u "${APIC_USER}" -p "${APIC_PASS}"

# apic build: 프로젝트 자산을 툴킷 포맷 아카이브로 생성
# -l 은 로컬 디렉토리(프로젝트들이 있는 상위 폴더)
# FILE은 빌드할 프로젝트/자산(여기선 apic_cicd 폴더)
apic build -l "${WORK}" -o "${ARCHIVE}" apic_cicd

ls -lh "${ARCHIVE}" || true

echo "=== Publishing built archive ==="
if [ "${USE_SPACE}" = "true" ] && [ -n "${SPACE_NAME}" ]; then
  apic projects:publish \
    --insecure-skip-tls-verify \
    --server "${APIC_SERVER}" \
    --org "${APIC_ORG}" \
    --catalog "${APIC_CATALOG}" \
    --scope space \
    --gateway_services "${GATEWAY_SERVICES}" \
    --migrate_subscriptions \
    "${ARCHIVE}"
else
  apic projects:publish \
    --insecure-skip-tls-verify \
    --server "${APIC_SERVER}" \
    --org "${APIC_ORG}" \
    --catalog "${APIC_CATALOG}" \
    --gateway_services "${GATEWAY_SERVICES}" \
    --migrate_subscriptions \
    "${ARCHIVE}"
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