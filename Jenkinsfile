pipeline {
    agent any

    environment {
        IMAGE = "mzaygar/apache2:latest"
        LOG_DIR = "scan_logs"
    }

    stages {
        // Etapa de preparar logs
        stage('Preparar logs') {
            steps {
                sh 'mkdir -p ${LOG_DIR}'
            }
        }

        // Verificar si Docker está accesible
        stage('Verificar acceso a Docker') {
            steps {
                script {
                    if (fileExists('/var/run/docker.sock')) {
                        echo 'Docker socket encontrado.'
                        sh 'sudo usermod -aG docker jenkins'
                    } else {
                        error 'No se encontró el socket de Docker.'
                    }
                }
            }
        }

        // Obtener digest de la imagen Docker
        stage('Obtener Digest') {
            steps {
                script {
                    sh "docker pull ${IMAGE}"
                    def digest = sh(script: "docker inspect --format='{{index .RepoDigests 0}}' ${IMAGE}", returnStdout: true).trim()
                    env.IMAGE_DIGEST = digest
                    echo "Digest obtenido: ${digest}"
                }
            }
        }

        // Escanear la imagen con Trivy
        stage('Escanear con Trivy') {
            steps {
                sh """
                    trivy image --scanners vuln --vuln-type os --severity CRITICAL,HIGH ${IMAGE} | tee ${LOG_DIR}/trivy_scan.log
                """
            }
        }

        // Firmar y verificar imagen con Cosign
        stage('Firmar y Verificar con Cosign') {
            environment {
                CONTRASENA_COSIGN = credentials('COSIGN_PASSWORD')
            }
            steps {
                withCredentials([ 
                    file(credentialsId: 'COSIGN_KEY', variable: 'COSIGN_KEY'),
                    file(credentialsId: 'COSIGN_PUB', variable: 'COSIGN_PUB')
                ]) {
                    sh """
                        COSIGN_PASSWORD=$CONTRASENA_COSIGN cosign sign --key $COSIGN_KEY ${IMAGE} | tee ${LOG_DIR}/cosign_sign.log
                        cosign verify --key $COSIGN_PUB ${IMAGE} | tee ${LOG_DIR}/cosign_verify.log
                        cosign verify --key $COSIGN_PUB ${IMAGE_DIGEST} | tee ${LOG_DIR}/cosign_digest_verify.log
                    """
                }
            }
        }

        // Desplegar imagen Docker
        stage('Desplegar Imagen en Docker') {
            steps {
                sh '''
                    docker rm -f apache2-container || true
                    docker pull $IMAGE_DIGEST
                    docker run -d --name apache2-container -p 8081:80 $IMAGE_DIGEST
                '''
            }
        }

        // Instalar Falco via Helm
        stage('Instalar Falco via Helm') {
            steps {
                retry(3) {
                    sh 'helm repo add falcosecurity https://falcosecurity.github.io/charts'
                    sh 'helm repo update'
                    sh 'helm install --replace falco --namespace falco --create-namespace --set tty=true falcosecurity/falco'
                }
            }
        }

        // Simular alerta Falco
        stage('Simular Alerta Falco') {
            steps {
                sh 'echo "Simulando comportamiento sospechoso..."'
                sh 'touch /bin/evil_script.sh || true'
            }
        }

        // Revisar logs de Falco
        stage('Revisar Logs Falco') {
            steps {
                sh 'kubectl logs -l app=falco -n falco --tail=100'
            }
        }

        // Archivar logs
        stage('Archivar Logs en Jenkins') {
            steps {
                archiveArtifacts artifacts: "${LOG_DIR}/*.log", fingerprint: true
            }
        }

        // Copiar logs a carpeta del servidor
        stage('Copiar logs a carpeta del servidor') {
            steps {
                sh "cp -r ${LOG_DIR} /var/logs/jenkins_logs/"
            }
        }
    }
}
