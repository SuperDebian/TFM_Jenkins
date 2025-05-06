pipeline {
    agent any

    environment {
        IMAGE_NAME = 'my-wordpress'
        IMAGE_TAG = 'latest'
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USER = 'mzaygar'
        DOCKER_PASS = credentials('docker-password')  // Asegúrate de que la credencial esté bien configurada
        COSIGN_KEY = credentials('cosign-key')  // Asegúrate de que la credencial esté bien configurada
        WORDPRESS_IMAGE = 'wordpress:latest'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/SuperDebian/TFM_Jenkins.git'
            }
        }

        stage('Pull WordPress Image') {
            steps {
                script {
                    // Usar Docker para obtener la imagen de WordPress
                    sh "docker pull ${WORDPRESS_IMAGE}"
                }
            }
        }

        stage('Scan with Trivy') {
            steps {
                script {
                    // Escanear la imagen con Trivy
                    sh "trivy image ${WORDPRESS_IMAGE}"
                }
            }
        }

        stage('Sign Image with Cosign') {
            steps {
                script {
                    // Firmar la imagen con Cosign
                    sh "cosign sign --key ${COSIGN_KEY} ${DOCKER_REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    // Login a Docker y subir la imagen
                    sh """
                        echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin
                        docker tag ${WORDPRESS_IMAGE} ${DOCKER_REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                // Limpiar los contenedores de Docker y liberar espacio
                sh 'docker system prune -f'
            }
        }
    }
}
