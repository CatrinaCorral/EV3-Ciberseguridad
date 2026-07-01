pipeline {
  agent any

  environment {
    APP_URL     = 'http://localhost:5000'
    SONAR_HOST = 'http://sonarqube-custom:9000'
    REPORT_DIR = 'zap-reports'
    VENV       = 'venv'
  }

  stages {

    stage('Build') {
      steps {
        sh '''
          docker network create devsecops-network || true
          docker build -t ev3-ciberseguridad:${BUILD_NUMBER} .
        '''
      }
    }

    stage('Test - Unit') {
      steps {
        sh '''
          python3 -m venv ${VENV} || true
          . ${VENV}/bin/activate || true

          pip install --upgrade pip || true
          pip install -r requirements.txt || true
          pip install pytest || true

          pytest --junitxml=results.xml || true
        '''

        junit allowEmptyResults: true, testResults: 'results.xml'
      }
    }

    stage('Deploy - Staging') {
      steps {
        sh '''
          docker rm -f ev3-app || true

          docker run -d \
            --name ev3-app \
            --network devsecops-network \
            -p 5000:5000 \
            ev3-ciberseguridad:${BUILD_NUMBER}

          sleep 5

          docker run --rm \
            --network devsecops-network \
            curlimages/curl:latest \
            -sf http://ev3-app:5000 || echo "App no responde"
        '''
      }
    }

    stage('Analyze - SonarQube') {
      steps {
        withSonarQubeEnv('sonarqube-server') {
          withEnv(["PATH+SONAR=${tool 'sonarqube-scanner'}/bin"]) {
            sh """
              sonar-scanner \
                -Dsonar.projectKey=ev3-ciberseguridad \
                -Dsonar.projectName=EV3-Ciberseguridad \
                -Dsonar.sources=. \
                -Dsonar.python.version=3 \
                -Dsonar.exclusions=venv/**,zap-reports/**,dc-report/**
            """
          }
        }
      }
    }

    stage('Security Test - SCA Dependencies') {
      steps {
        sh '''
          . ${VENV}/bin/activate || true

          pip install bandit safety || true

          echo "===== RESULTADOS SECURITY SCAN - BANDIT ====="
          bandit -r . -x ./venv -f txt || true

          echo "===== RESULTADOS DEPENDENCY SCAN - SAFETY ====="
          safety check -r requirements.txt || true
        '''
      }
    }

    stage('Security Test - DAST ZAP') {
      steps {
        sh """
          rm -rf \${WORKSPACE}/zap-reports
          mkdir -p \${WORKSPACE}/zap-reports
          chmod 777 \${WORKSPACE}/zap-reports

          docker run --rm \
            --network devsecops-network \
            -v \${WORKSPACE}/zap-reports:/zap/wrk/:rw \
            ghcr.io/zaproxy/zaproxy:stable \
              zap-baseline.py \
                -t http://ev3-app:5000 \
                -r zap_report.html \
                -J zap_report.json \
                --auto || true

          echo "=== Contenido zap-reports ==="
          ls -la \${WORKSPACE}/zap-reports/
        """

        publishHTML(target: [
          allowMissing         : true,
          alwaysLinkToLastBuild: true,
          keepAll              : true,
          reportDir            : "${WORKSPACE}/zap-reports",
          reportFiles          : 'zap_report.html',
          reportName           : 'OWASP ZAP Report'
        ])

        sh 'find ${WORKSPACE} -name "*.html" -o -name "*.json" 2>/dev/null | head -30'
      }
    }

    stage('Deploy') {
      steps {
        echo 'Despliegue finalizado en entorno de prueba.'
      }
    }
  }

  post {
    always {
      sh """
        cp \${WORKSPACE}/zap-reports/*.html \${WORKSPACE}/ 2>/dev/null || true
        cp \${WORKSPACE}/zap-reports/*.json \${WORKSPACE}/ 2>/dev/null || true
      """

      archiveArtifacts(
        artifacts         : 'zap_report.html,zap_report.json,results.xml',
        allowEmptyArchive : true
      )

      sh 'docker rm -f ev3-app || true'
    }

    success {
      echo 'Pipeline ejecutado correctamente.'
    }

    failure {
      echo 'Pipeline fallo.'
    }
  }
}
