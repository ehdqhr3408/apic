pipeline {
  agent any

  environment {
    // apic가 설치된 노드 (k8s-worker2 IP)
    DEPLOY_HOST = "10.0.0.80"

    // APIC 접속정보
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

# 소스 전송
ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} "rm -rf ${WORK} && mkdir -p ${WORK}"
scp -o StrictHostKeyChecking=no -r apic_cicd root@${DEPLOY_HOST}:${WORK}/

# 배포 실행 (원격에서도 bash 강제, 변수 확장 분리)
ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} bash -lc "
  set -euo pipefail
  cd '${WORK}/apic_cicd'

  apic version
  apic login --server '${APIC_SERVER}' --realm '${APIC_REALM}' -u '${APIC_USER}' -p '${APIC_PASS}'

  ls -1 *_prd.yml >/dev/null 2>&1 || { echo 'NO *_prd.yml found'; exit 1; }

  for f in *_prd.yml; do
    echo \"=== Publishing \$f ===\"
    apic products:publish \"\$f\" --server '${APIC_SERVER}' --org '${APIC_ORG}' --catalog '${APIC_CATALOG}'
  done
"

# 정리
ssh -o StrictHostKeyChecking=no root@${DEPLOY_HOST} "rm -rf ${WORK}"
'''
          }
        }
      }
    }
  }
}