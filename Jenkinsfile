pipeline {
    agent any

    stages {
        stage('Construccion') {
            steps {
                echo 'Construyendo la aplicacion vulnerable...'
            }
        }

        stage('Security Scan') {
            steps {
                echo 'Ejecutando revision de seguridad sobre vulnerable_app.py...'

                sh '''
                    echo "Hallazgo SQL Injection:"
                    grep -n "SELECT \\* FROM users WHERE username" vulnerable_app.py || true

                    echo "Hallazgo XSS:"
                    grep -n "task = request.form\\['task'\\]" vulnerable_app.py || true
                    grep -n "task\\['task'\\]" vulnerable_app.py || true

                    echo "Hallazgo CSRF:"
                    grep -n "@app.route('/delete_task" vulnerable_app.py || true
                '''
            }
        }

        stage('Pruebas') {
            steps {
                echo 'Ejecutando pruebas basicas...'
                echo 'Verificando archivos del proyecto'
            }
        }

        stage('Despliegue') {
            steps {
                echo 'Desplegando aplicacion en entorno de prueba...'
            }
        }
    }
}