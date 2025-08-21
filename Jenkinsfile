pipeline {
  agent any

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  tools {
    jdk   'jdk11'   // Spring Boot 2.7.x + java.version=11
    maven 'maven3'
  }

  environment {
    // === Repositorio ===
    GIT_URL    = 'git@github.com:cachitomevi/06_springboot-ci-staging.git'
    GIT_BRANCH = 'master'

    // === Artefacto ===
    ARTIFACT_NAME = 'demo-0.0.1-SNAPSHOT.jar'

    // === Staging ===
    SSH_CRED     = 'staging_ssh'            // <-- credencial SSH para tu servidor
    STAGING_USER = 'deploy'             // <-- CAMBIAR
    STAGING_HOST = 'localhost'           // <-- CAMBIAR
    STAGING_DIR  = '/home/deploy/staging' // <-- CAMBIAR
    STAGING_PORT = '8081'

    HEALTH_URL = "http://${STAGING_HOST}:${STAGING_PORT}/health"
  }

  stages {

    stage('Checkout') {
      steps {
        // Clona por SSH usando la credencial de GitHub
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
          junit 'target/surefire-reports/*.xml'
          archiveArtifacts artifacts: 'target/site/jacoco/**', allowEmptyArchive: true
        }
      }
    }

    stage('Code Quality (Checkstyle)') {
      steps {
        // Requiere el plugin de Checkstyle configurado en el POM
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
        // Usa la credencial SSH del servidor de Staging
        sshagent (credentials: [env.SSH_CRED]) {
          sh '''
            set -euo pipefail

            # 1) Asegura carpeta remota
            ssh -o StrictHostKeyChecking=no ${STAGING_USER}@${STAGING_HOST} "mkdir -p ${STAGING_DIR}"

            # 2) Sube el jar compilado
            scp -o StrictHostKeyChecking=no "target/${ARTIFACT_NAME}" ${STAGING_USER}@${STAGING_HOST}:${STAGING_DIR}/app.jar

            # 3) Reinicia la app con PID file
            ssh -o StrictHostKeyChecking=no ${STAGING_USER}@${STAGING_HOST} bash -lc '
              set -euo pipefail
              cd ${STAGING_DIR}

              if [ -f app.pid ] && kill -0 $(cat app.pid) 2>/dev/null; then
                echo "Deteniendo proceso previo PID=$(cat app.pid)"
                kill $(cat app.pid) || true
                sleep 3
              fi

              nohup java -jar app.jar --server.port=${STAGING_PORT} > app.log 2>&1 &
              echo $! > app.pid
              echo "Nuevo PID: $(cat app.pid)"
            '
          '''
        }
      }
    }

    stage('Validate Deployment') {
      steps {
        sh '''
          set -e
          echo "Health check: ${HEALTH_URL}"
          for i in $(seq 1 20); do
            if curl -fsS "${HEALTH_URL}" | grep -q "OK"; then
              echo "Servicio OK en ${HEALTH_URL}"
              exit 0
            fi
            echo "Intento $i/20... esperando 3s"
            sleep 3
          done
          echo "Health check FAILED"
          curl -i "${HEALTH_URL}" || true
          exit 1
        '''
      }
    }
  }

  post {
    success { echo '✅ OK: tests, cobertura, checkstyle, package, deploy y health check (8081).' }
    failure { echo '❌ Falló el pipeline. Revisa el stage en rojo y los logs.' }
    always  {
      archiveArtifacts artifacts: "target/${ARTIFACT_NAME}, target/site/jacoco/**, target/checkstyle-result.xml",
                       allowEmptyArchive: true
    }
  }
}
