pipeline {
  agent any

  environment {
    APP_URL       = 'http://localhost:5000'
    SONAR_HOST    = 'http://sonarqube-custom:9000/'
    REPORT_DIR    = 'zap-reports'
    DC_HOME       = tool 'dependency-check'
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
          python3 -m venv venv || true
          . venv/bin/activate || true

          pip install --upgrade pip || true
          pip install -r requirements.txt || true
          pip install pytest flask || true

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
                -Dsonar.projectKey=EV3-Ciberseguridad \
                -Dsonar.projectName=EV3-Ciberseguridad \
                -Dsonar.sources=. \
                -Dsonar.python.version=3 \
                -Dsonar.exclusions=venv/**,zap-reports/**,dc-report/** \
                -Dsonar.token=$SONAR_AUTH_TOKEN
            """
          }
        }
      }
    }

    stage('Security Test - SCA Dependencies') {
      steps {
        sh """
          docker volume rm dc-report-vol 2>/dev/null || true
          docker volume create dc-report-vol
          docker volume create dc-nvd-data || true

          docker run --rm \
            --network devsecops-network \
            -v \${WORKSPACE}:/src:ro \
            -v dc-report-vol:/report \
            -v dc-nvd-data:/usr/share/dependency-check/data \
            owasp/dependency-check:latest \
              --scan /src \
              --format HTML \
              --format XML \
              --out /report \
              --project EV3-Ciberseguridad \
              --noupdate || true

          docker run --rm \
            -v dc-report-vol:/report \
            alpine ls -la /report/

          docker run --rm \
            -v dc-report-vol:/report \
            alpine cat /report/dependency-check-report.xml || true

          mkdir -p \${WORKSPACE}/dc-report
        """

        sh '''
          docker run --rm \
            -v dc-report-vol:/report \
            -v ${WORKSPACE}/dc-report:/dest \
            alpine sh -c 'cp /report/*.html /dest/ 2>/dev/null || true; cp /report/*.xml /dest/ 2>/dev/null || true; chmod 644 /dest/* 2>/dev/null || true'

          echo "=== Archivos copiados al workspace ==="
          ls -la ${WORKSPACE}/dc-report/ || true
        '''

        publishHTML(target: [
          allowMissing         : true,
          alwaysLinkToLastBuild: true,
          keepAll              : true,
          reportDir            : "${WORKSPACE}/dc-report",
          reportFiles          : 'dependency-check-report.html',
          reportName           : 'Dependency-Check Report'
        ])
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

        sh 'find ${WORKSPACE} -name "*.html" -o -name "*.xml" -o -name "*.json" 2>/dev/null | head -30'
      }
    }
  }

  post {
    always {
      sh """
        cp \${WORKSPACE}/dc-report/*.html \${WORKSPACE}/ 2>/dev/null || true
        cp \${WORKSPACE}/dc-report/*.xml \${WORKSPACE}/ 2>/dev/null || true
        cp \${WORKSPACE}/zap-reports/*.html \${WORKSPACE}/ 2>/dev/null || true
        cp \${WORKSPACE}/zap-reports/*.json \${WORKSPACE}/ 2>/dev/null || true
      """

      archiveArtifacts(
        artifacts         : 'dependency-check-report.html,dependency-check-report.xml,zap_report.html,zap_report.json,results.xml',
        allowEmptyArchive : true
      )

      sh 'docker rm -f ev3-app || true'
    }
  }
}
