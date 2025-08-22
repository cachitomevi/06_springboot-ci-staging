pipeline {
  agent any

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  tools {
    jdk   'jdk11'
    maven 'maven3'
  }

  environment {
    // === Repositorio ===
    GIT_URL    = 'git@github.com:cachitomevi/06_springboot-ci-staging.git'
    GIT_BRANCH = 'master'

    // === Artefacto ===
    ARTIFACT_NAME = 'demo-0.0.1-SNAPSHOT.jar'

    // === Staging (WSL) ===
    SSH_CRED     = 'staging_ssh'
    STAGING_USER = 'deploy'
    STAGING_HOST = '172.22.228.104'   // IP WSL
    SSH_PORT     = '22'
    STAGING_DIR  = '/home/deploy/staging'
    STAGING_PORT = '8081'

    HEALTH_URL = "http://${STAGING_HOST}:${STAGING_PORT}/health"
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: "${GIT_BRANCH}",
            url: "${GIT_URL}",
            credentialsId: 'github_ssh_key'
      }
    }

    stage('Build & Tests (verify + JaCoCo)') {
      steps {
        sh 'mvn -B -DskipTests=false clean verify'
      }
      post {
        always {
          junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
          archiveArtifacts artifacts: 'target/site/jacoco/**', allowEmptyArchive: true
        }
      }
    }

    stage('Code Quality (Checkstyle)') {
      steps {
        sh 'mvn -B checkstyle:checkstyle'
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/checkstyle-result.xml', allowEmptyArchive: true
        }
      }
    }

    stage('Package') {
      steps {
        sh 'mvn -B -DskipTests package'
        archiveArtifacts artifacts: "target/${ARTIFACT_NAME}", fingerprint: true
      }
    }

    stage('Deploy to Staging (SSH)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: env.SSH_CRED,
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          // Nota: usamos 'set -eu' (sin pipefail) para compatibilidad POSIX
          sh '''#!/usr/bin/env bash
set -eu

# 1) Prepara carpeta remota
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i "$SSH_KEY" -p "${SSH_PORT}" \
  "$SSH_USER@${STAGING_HOST}" "mkdir -p ${STAGING_DIR}"

# 2) Sube el jar compilado
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i "$SSH_KEY" -P "${SSH_PORT}" \
  "target/${ARTIFACT_NAME}" "$SSH_USER@${STAGING_HOST}:${STAGING_DIR}/app.jar"

# 3) Reinicia la app en remoto (manejo seguro de PID)
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i "$SSH_KEY" -p "${SSH_PORT}" \
  "$SSH_USER@${STAGING_HOST}" bash -lc "
  set -eu
  cd ${STAGING_DIR}

  if [ -f app.pid ]; then
    pid=\\$(cat app.pid || true)
    if [ -n \\\"\\$pid\\\" ] && kill -0 \\\"\\$pid\\\" 2>/dev/null; then
      echo \\\"Deteniendo proceso previo PID=\\$pid\\\"
      kill \\\"\\$pid\\\" || true
      sleep 3
    else
      rm -f app.pid
    fi
  fi

  nohup java -jar app.jar --server.port=${STAGING_PORT} > app.log 2>&1 &
  echo \\$! > app.pid
  echo \\\"Nuevo PID: \\$(cat app.pid)\\\"
"
'''
        }
      }
    }

    stage('Validate Deployment') {
  steps {
    withCredentials([sshUserPrivateKey(credentialsId: env.SSH_CRED,
                                       keyFileVariable: 'SSH_KEY',
                                       usernameVariable: 'SSH_USER')]) {
      sh '''#!/usr/bin/env bash
set -eu

echo "Health check remoto en WSL: ${HEALTH_URL}"

# Probar 20 veces contra localhost dentro de WSL
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i "$SSH_KEY" -p "${SSH_PORT}" \
  "$SSH_USER@${STAGING_HOST}" bash -lc '
    set -eu
    echo "Probando salud en localhost: ${HEALTH_URL}"
    for i in $(seq 1 20); do
      if curl -fsS "http://127.0.0.1:${STAGING_PORT}/health" | grep -q "OK"; then
        echo "Servicio OK en 127.0.0.1:${STAGING_PORT}/health"
        exit 0
      fi
      echo "Intento $i/20... esperando 3s"
      sleep 3
    done
    echo "Health check FAILED"
    echo "---- Tail de app.log ----"
    tail -n 200 "${STAGING_DIR}/app.log" || true
    exit 1
  '
'''
    }
  }
}


  post {
    success {
      echo '✅ OK: tests, cobertura, checkstyle, package, deploy y health check (8081).'
    }
    failure {
      echo '❌ Falló el pipeline. Revisa el stage en rojo y los logs.'
    }
    always  {
      archiveArtifacts artifacts: "target/${ARTIFACT_NAME}, target/site/jacoco/**, target/checkstyle-result.xml",
                       allowEmptyArchive: true
    }
  }
}
