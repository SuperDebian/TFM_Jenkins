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
                    trivy image --scanners vuln --vuln-type os --severity CRITICAL,HIGH,MEDIUM,LOW ${IMAGE} | tee ${LOG_DIR}/trivy_scan.log
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
                    script {
                        def imageDigest = sh(script: "docker inspect --format='{{index .RepoDigests 0}}' ${IMAGE}", returnStdout: true).trim()

                        sh """
                            export COSIGN_PASSWORD=\$CONTRASENA_COSIGN
                            cosign sign --yes --key \$COSIGN_KEY ${imageDigest} | tee ${LOG_DIR}/cosign_sign.log
                            cosign verify --key \$COSIGN_PUB ${imageDigest} | tee ${LOG_DIR}/cosign_verify.log
                        """
                    }
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

        stage('Archivar Logs en Jenkins') {
            steps {
                archiveArtifacts artifacts: "${LOG_DIR}/*.log", fingerprint: true, allowEmptyArchive: true
            }
        }

        stage('Copiar logs a carpeta del servidor') {
            steps {
                sh '''
                    echo "Copiando logs localmente..."
                    sudo mkdir -p /var/logs/jenkins_logs/
                    sudo cp -r ${LOG_DIR}/* /var/logs/jenkins_logs/
                '''
            }
        }
    }
}
