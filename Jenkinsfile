pipeline {
    agent any

    environment {
        IMAGE_NAME = 'my-wordpress'
        IMAGE_TAG = 'latest'
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USER = 'mzaygar'
        DOCKER_PASS = credentials('docker-password')
        COSIGN_KEY = credentials('cosign-key')
        WORDPRESS_IMAGE = 'wordpress:latest'
    }

    stages {
        stage('Install Tools') {
            steps {
                script {
                    sh '''
                        echo "[*] Instalando Docker, Trivy y Cosign..."
                        apt-get update && apt-get install -y docker.io curl git wget

                        # Instalar Trivy
                        wget -q https://github.com/aquasecurity/trivy/releases/download/v0.49.1/trivy_0.49.1_Linux-64bit.deb
                        dpkg -i trivy_0.49.1_Linux-64bit.deb
                        rm -f trivy_0.49.1_Linux-64bit.deb

                        # Instalar Cosign
                        wget -q https://github.com/sigstore/cosign/releases/download/v2.4.1/cosign-linux-amd64
                        chmod +x cosign-linux-amd64
                        mv cosign-linux-amd64 /usr/local/bin/cosign
                    '''
                }
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/SuperDebian/TFM_Jenkins.git'
            }
        }

        stage('Pull WordPress Image') {
            steps {
                sh "docker pull ${WORDPRESS_IMAGE}"
            }
        }

        stage('Scan with Trivy') {
            steps {
                sh "trivy image ${WORDPRESS_IMAGE}"
            }
        }

        stage('Sign Image with Cosign') {
            steps {
                sh "cosign sign --key ${COSIGN_KEY} ${DOCKER_REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                    echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin
                    docker tag ${WORDPRESS_IMAGE} ${DOCKER_REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${DOCKER_REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
    }

    post {
        always {
            sh 'docker system prune -f'
        }
    }
}
