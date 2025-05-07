pipeline {
    agent any

    environment {
        IMAGE = "mzaygar/apache2:latest"
        LOG_DIR = "scan_logs"
    }

    stages {
        stage('Preparar logs') {
            steps {
                sh 'mkdir -p ${LOG_DIR}'
            }
        }

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

        stage('Escanear con Trivy') {
            steps {
                sh """
                    trivy image --scanners vuln --vuln-type os --severity CRITICAL,HIGH ${IMAGE} | tee ${LOG_DIR}/trivy_scan.log
                """
            }
        }

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

        stage('Desplegar Imagen en Docker') {
            steps {
                sh '''
                    docker rm -f apache2-container || true
                    docker pull $IMAGE_DIGEST
                    docker run -d --name apache2-container -p 8081:80 $IMAGE_DIGEST
                '''
            }
        }

        // Etapas de Falco
        stage('Instalar Falco via Helm') {
            steps {
                sh 'helm repo add falcosecurity https://falcosecurity.github.io/charts'
                sh 'helm repo update'
                sh 'helm install falco falcosecurity/falco'
            }
        }

        stage('Simular Alerta Falco') {
            steps {
                sh 'echo "Simulando comportamiento sospechoso..."'
                sh 'touch /bin/evil_script.sh || true'
            }
        }

        stage('Revisar Logs Falco') {
            steps {
                sh 'kubectl logs -l app=falco -n falco --tail=100'
            }
        }

        stage('Simular Alerta de Seguridad') {
            steps {
                sh '''
                    echo "Simulando comportamiento sospechoso en contenedor..."
                    docker exec apache2-container sh -c "touch /bin/evil_script.sh || true"
                '''
            }
        }

        stage('Revisar Logs del Contenedor') {
            steps {
                sh 'docker logs apache2-container --tail 100 | tee ${LOG_DIR}/docker_logs.log'
            }
        }

        stage('Archivar Logs en Jenkins') {
            steps {
                archiveArtifacts artifacts: "${LOG_DIR}/*.log", fingerprint: true
            }
        }

        stage('Copiar logs a carpeta del servidor') {
            steps {
                sh "cp -r ${LOG_DIR} /var/logs/jenkins_logs/"
            }
        }
    }
}
