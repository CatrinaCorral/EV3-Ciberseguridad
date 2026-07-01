pipeline {
    agent any

    environment {
        VENV = 'venv'
        REPORT_DIR = 'zap-reports'
    }

    stages {
        stage('Construccion') {
            steps {
                echo 'Construyendo la aplicacion vulnerable...'
                echo 'Repositorio descargado desde GitHub'
            }
        }

        stage('Python Version') {
            steps {
                sh 'python3 --version || true'
                sh 'pip3 --version || true'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv ${VENV} || true
                    . ${VENV}/bin/activate || true

                    pip install --upgrade pip || true
                    pip install -r requirements.txt || true
                    pip install bandit safety pytest || true
                '''
            }
        }

        stage('Security Scan') {
            steps {
                echo 'Ejecutando analisis de seguridad automatizado con Bandit y Safety...'

                sh '''
                    . ${VENV}/bin/activate || true

                    echo "===== RESULTADOS SECURITY SCAN - BANDIT ====="
                    bandit -r . -x ./venv -f txt || true

                    echo "===== RESULTADOS DEPENDENCY SCAN - SAFETY ====="
                    safety check -r requirements.txt || true
                '''
            }
        }

        stage('Deploy - Staging') {
            steps {
                echo 'Desplegando aplicacion en entorno de prueba para OWASP ZAP...'

                sh '''
                    docker network create ev3-network || true

                    docker rm -f ev3-app || true

                    docker build -t ev3-ciberseguridad:${BUILD_NUMBER} .

                    docker run -d \
                        --name ev3-app \
                        --network ev3-network \
                        -p 5000:5000 \
                        ev3-ciberseguridad:${BUILD_NUMBER}

                    sleep 10

                    docker run --rm \
                        --network ev3-network \
                        curlimages/curl:latest \
                        -s http://ev3-app:5000 || echo "App no responde"
                '''
            }
        }

        stage('Security Test - DAST ZAP') {
            steps {
                echo 'Ejecutando pruebas automatizadas de seguridad con OWASP ZAP...'

                sh '''
                    rm -rf ${WORKSPACE}/${REPORT_DIR}
                    mkdir -p ${WORKSPACE}/${REPORT_DIR}
                    chmod 777 ${WORKSPACE}/${REPORT_DIR}

                    docker run --rm \
                        --network ev3-network \
                        -v ${WORKSPACE}/${REPORT_DIR}:/zap/wrk/:rw \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t http://ev3-app:5000 \
                        -r zap_report.html \
                        -J zap_report.json \
                        -I || true

                    echo "===== ARCHIVOS GENERADOS POR OWASP ZAP ====="
                    ls -la ${WORKSPACE}/${REPORT_DIR}
                '''
            }
        }

        stage('Pruebas') {
            steps {
                echo 'Ejecutando pruebas basicas...'
                echo 'Verificando archivos del proyecto'

                sh '''
                    . ${VENV}/bin/activate || true
                    pytest || true
                '''
            }
        }

        stage('Despliegue') {
            steps {
                echo 'Desplegando aplicacion en entorno de prueba...'
            }
        }
    }

    post {
        always {
            echo 'Archivando reportes generados por OWASP ZAP...'

            archiveArtifacts artifacts: 'zap-reports/*', allowEmptyArchive: true

            sh '''
                docker rm -f ev3-app || true
            '''
        }

        success {
            echo 'Pipeline ejecutado correctamente'
        }

        failure {
            echo 'Pipeline fallo'
        }
    }
}
