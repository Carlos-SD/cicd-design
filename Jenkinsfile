#!/usr/bin/env groovy

pipeline {
    agent any

    environment {
        APP_NAME      = "cicd-demo"
        IMAGE_NAME    = "mi-app"
        IMAGE_TAG     = "latest"
        SONAR_HOST    = "http://sonarqube:9000"
        SONAR_PROJECT = "cicd-demo"
        SONAR_TOKEN   = credentials('SONAR_AUTH_TOKEN')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Static Analysis (SonarQube)') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=${SONAR_PROJECT} \
                          -Dsonar.host.url=${SONAR_HOST} \
                          -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline detenido por Quality Gate: estado=${qg.status}"
                        }
                        // Verify no open Security Hotspots remain
                        def hotspots = sh(
                            script: """
                                curl -s -u ${SONAR_TOKEN}: \
                                  "${SONAR_HOST}/api/hotspots/search?projectKey=${SONAR_PROJECT}&status=TO_REVIEW" \
                                | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('paging', {}).get('total', 0))"
                            """,
                            returnStdout: true
                        ).trim()
                        if (hotspots.toInteger() > 0) {
                            error "Pipeline detenido: ${hotspots} Security Hotspot(s) sin revisar en SonarQube"
                        }
                    }
                }
            }
        }

        stage('Container Security Scan (Trivy)') {
            steps {
                script {
                    def trivyExit = sh(
                        script: "trivy image --exit-code 1 --severity CRITICAL --no-progress --ignorefile .trivyignore ${IMAGE_NAME}:${IMAGE_TAG}",
                        returnStatus: true
                    )
                    if (trivyExit != 0) {
                        error "Pipeline detenido: Trivy encontró vulnerabilidades CRITICAL en la imagen ${IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Deploy') {
            when { branch 'master' }
            steps {
                script {
                    sh "docker stop ${APP_NAME} || true"
                    sh "docker rm   ${APP_NAME} || true"
                    sh "docker run -d --name ${APP_NAME} -p 80:8080 ${IMAGE_NAME}:${IMAGE_TAG}"
                    echo "Aplicación desplegada en http://localhost:80"
                }
            }
        }
    }

    post {
        always {
            echo 'Limpiando espacio de trabajo...'
            cleanWs()
        }
        success {
            echo "Pipeline completado exitosamente."
        }
        failure {
            echo "Pipeline FALLIDO. Revisa los logs anteriores para más detalles."
            // Para notificaciones por correo descomenta la siguiente línea
            // mail to: 'equipo@ejemplo.com', subject: "FALLO: ${env.JOB_NAME} #${env.BUILD_NUMBER}", body: "Ver: ${env.BUILD_URL}"
        }
    }
}
