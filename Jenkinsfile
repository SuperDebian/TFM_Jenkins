pipeline {
    agent any

    environment {
        IMAGE = "mzaygar/apache2:latest"
        LOG_DIR = "scan_logs"
    }

    stages {
        stage('Preparar logs') {
            steps {
                sh 'mkdir -p $LOG_DIR'
            }
        }

        stage('Obtener Digest') {
            steps {
                script {
                    def digest = sh(
                        script: "docker pull $IMAGE > /dev/null && docker inspect --format='{{index .RepoDigests 0}}' $IMAGE",
                        returnStdout: true
                    ).trim()
                    echo "Digest obtenido: ${digest}"
                    env.DIGEST = digest
                }
            }
        }

        stage('Escanear con Trivy') {
            steps {
                sh 'trivy image --scanners vuln --vuln-type os --severity CRITICAL,HIGH $IMAGE | tee $LOG_DIR/trivy_scan.log'
            }
        }

        stage('Firmar y Verificar con Cosign') {
            steps {
                withCredentials([ 
                    file(credentialsId: 'COSIGN_KEY', variable: 'COSIGN_KEY_FILE'),
                    file(credentialsId: 'COSIGN_PUB', variable: 'COSIGN_PUB_FILE')
                ]) {
                    sh '''
                        cosign sign -key $COSIGN_KEY_FILE $IMAGE | tee $LOG_DIR/cosign_sign.log
                        cosign verify --key $COSIGN_PUB_FILE $IMAGE | tee $LOG_DIR/cosign_verify.log
                        cosign verify --key $COSIGN_PUB_FILE $DIGEST | tee $LOG_DIR/cosign_digest_verify.log
                    '''
                }
            }
        }

        stage('Desplegar Imagen en Kubernetes') {
            steps {
                script {
                    // Crear el deployment en Kubernetes con el digest de la imagen
                    sh "kubectl create deployment apache2 --image=$DIGEST"

                    // Exponer el deployment como un servicio de tipo NodePort
                    sh "kubectl expose deployment apache2 --type=NodePort --port=80"

                    // Esperar un poco para asegurarse de que el servicio esté disponible
                    sleep(10)

                    // Obtener la URL del servicio expuesto
                    def service_url = sh(script: 'minikube service apache2 --url', returnStdout: true).trim()
                    echo "Servicio expuesto en: ${service_url}"
                }
            }
        }

        stage('Instalar Falco via Helm') {
            steps {
                sh '''
                    minikube start
                    helm repo add falcosecurity https://falcosecurity.github.io/charts
                    helm repo update
                    helm install --replace falco --namespace falco --create-namespace --set tty=true falcosecurity/falco
                    kubectl wait pods --for=condition=Ready --all -n falco --timeout=120s
                '''
            }
        }

        stage('Simular Alerta Falco') {
            steps {
                sh '''
                    kubectl create deployment nginx --image=nginx
                    sleep 10
                    POD=$(kubectl get pods --selector=app=nginx -o jsonpath="{.items[0].metadata.name}")
                    kubectl exec -it $POD -- cat /etc/shadow || true
                '''
            }
        }

        stage('Revisar Logs Falco') {
            steps {
                sh 'kubectl logs -l app.kubernetes.io/name=falco -n falco -c falco | grep Warning | tee $LOG_DIR/falco_warnings.log || true'
            }
        }

        stage('Archivar Logs en Jenkins') {
            steps {
                archiveArtifacts artifacts: "$LOG_DIR/*.log", fingerprint: true
            }
        }

        stage('Copiar logs a carpeta del servidor') {
            steps {
                sh 'mkdir -p /var/log/falco_logs && cp $LOG_DIR/*.log /var/log/falco_logs/'
            }
        }
    }
}
