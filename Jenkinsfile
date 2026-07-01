pipeline {
    agent any

    environment {
        VENV = 'venv'
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
                echo 'Ejecutando analisis de seguridad automatizado...'

                sh '''
                    . ${VENV}/bin/activate || true

                    echo "===== RESULTADOS SECURITY SCAN - BANDIT ====="
                    bandit -r . -x ./venv -f txt || true

                    echo "===== RESULTADOS DEPENDENCY SCAN - SAFETY ====="
                    safety check -r requirements.txt || true
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
        success {
            echo 'Pipeline ejecutado correctamente'
        }

        failure {
            echo 'Pipeline fallo'
        }
    }
}
