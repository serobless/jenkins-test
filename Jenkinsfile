pipeline {
    agent any
    
    tools {
        maven 'maven'
    }
    
    environment {
        PROJECT_NAME = "jenkins-test"
        SONARQUBE_URL = "http://sonarqube:9000"
        SONARQUBE_TOKEN = squ_192869129e71837c12c9c1fb3b0d1bbd498e71b5
        NVD_API_KEY = credentials('nvdApiKey')
        TARGET_URL = 'http://172.24.81.81:5000'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '===== Clonando repositorio ====='
                checkout scm
            }
        }
        
        stage('Setup Environment') {
            steps {
                echo '===== Configurando entorno Python ====='
                sh '''
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                echo '===== Ejecutando análisis de SonarQube ====='
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQubeScanner') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=${PROJECT_NAME} \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=${SONARQUBE_URL} \
                            -Dsonar.token=${SONARQUBE_TOKEN}
                        """
                    }
                }
            }
        }
        
        stage('Python Security Audit') {
            steps {
                echo '===== Ejecutando pip-audit ====='
                sh '''
                    . .venv/bin/activate
                    pip install pip-audit
                    pip-audit -r requirements.txt --format json > pip-audit-report.json || true
                    pip-audit -r requirements.txt > pip-audit-report.txt || true
                '''
            }
        }
        
        stage('OWASP Dependency-Check') {
            steps {
                echo '===== Ejecutando OWASP Dependency-Check ====='
                dependencyCheck additionalArguments: """
                    --scan ./ \
                    --format HTML \
                    --format XML \
                    --format JSON \
                    --out dependency-check-report \
                    --nvdApiKey ${NVD_API_KEY}
                """, odcInstallation: 'DependencyCheck'
                
                dependencyCheckPublisher pattern: 'dependency-check-report/dependency-check-report.xml'
            }
        }
        
        stage('Publish Reports') {
            steps {
                echo '===== Publicando reportes HTML ====='
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'dependency-check-report',
                    reportFiles: 'dependency-check-report.html',
                    reportName: 'OWASP Dependency-Check Report'
                ])
            }
        }
    }
    
    post {
        always {
            echo '===== Pipeline finalizado ====='
            cleanWs()
        }
        success {
            echo '✅ Pipeline ejecutado exitosamente!'
        }
        failure {
            echo '❌ Pipeline falló. Revisa los logs.'
        }
    }
}
