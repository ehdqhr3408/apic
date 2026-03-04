pipeline {
  agent any

  environment {
    DEPLOY_HOST = "10.0.0.80"

    APIC_SERVER  = "https://manager.apic12.e1.com"
    APIC_REALM   = "provider/default-idp-2"
    APIC_ORG     = "enterprise1"
    APIC_CATALOG = "skt"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Publish Products') {
      steps {
        sshagent(credentials: ['ssh-k8s-worker2']) {
          withCredentials([usernamePassword(credentialsId: 'apic-cred',
                           usernameVariable: 'APIC_USER',
                           passwordVariable: 'APIC_PASS')]) {

            sh '''#!/usr/bin/env bash
set -euo pipefail

WORK="/tmp/apic_cicd_${BUILD_TAG}"

# 원격 작업 폴더 준비
ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} "rm -rf '${WORK}' && mkdir -p '${WORK}'"

# 소스 전송
scp -o StrictHostKeyChecking=no -r apic_cicd root@${DEPLOY_HOST}:"${WORK}/"

# 원격에서 bash로 실행 (로컬 변수 확장 충돌 방지)
ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} \
  env WORK="${WORK}" \
      APIC_SERVER="${APIC_SERVER}" \
      APIC_REALM="${APIC_REALM}" \
      APIC_ORG="${APIC_ORG}" \
      APIC_CATALOG="${APIC_CATALOG}" \
      APIC_USER="${APIC_USER}" \
      APIC_PASS="${APIC_PASS}" \
  bash -s <<'EOS'
set -euo pipefail

cd "${WORK}/apic_cicd"

apic version
apic login --server "${APIC_SERVER}" --realm "${APIC_REALM}" -u "${APIC_USER}" -p "${APIC_PASS}"

ls -1 *_prd.yml >/dev/null 2>&1 || { echo "NO *_prd.yml found"; exit 1; }

for p in *_prd.yml; do
  echo "=== Publishing ${p} ==="
  apic products:publish "${p}" --server "${APIC_SERVER}" --org "${APIC_ORG}" --catalog "${APIC_CATALOG}"
done
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