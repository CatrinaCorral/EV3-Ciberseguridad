pipeline {
  agent any

  environment {
    APP_URL    = 'http://localhost:5000'
    REPORT_DIR = 'zap-reports'
    VENV       = 'venv'
  }

  stages {

    stage('Build') {
      steps {
        sh 'docker build -t ev3-ciberseguridad:${BUILD_NUMBER} .'
      }
    }

    stage('Install Dependencies') {
      steps {
        sh '''
          python3 -m venv ${VENV} || true
          . ${VENV}/bin/activate || true

          pip install --upgrade pip || true
          pip install -r requirements.txt || true
          pip install pytest bandit safety || true
        '''
      }
    }

    stage('Test - Unit') {
      steps {
        sh '''
          . ${VENV}/bin/activate || true
          pytest || true
        '''
      }
    }

    stage('Deploy - Staging') {
      steps {
        sh '''
          docker rm -f ev3-app || true
          docker run -d --name ev3-app -p 5000:5000 ev3-ciberseguridad:${BUILD_NUMBER}
          sleep 5
          curl -sf http://localhost:5000 || echo "App no responde"
        '''
      }
    }

    stage('Security Test - SCA Dependencies') {
      steps {
        sh '''
          . ${VENV}/bin/activate || true

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
            --network host \
            -v \${WORKSPACE}/zap-reports:/zap/wrk/:rw \
            ghcr.io/zaproxy/zaproxy:stable \
              zap-baseline.py \
                -t http://localhost:5000 \
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
        artifacts         : 'zap_report.html,zap_report.json',
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
