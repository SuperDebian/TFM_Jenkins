pipeline {
    agent any

    environment {
        IMAGE_NAME = 'my-wordpress'  // Nombre de la imagen que subiremos
        IMAGE_TAG = 'latest'         // Etiqueta de la imagen
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USER = 'mzaygar'      // Tu usuario de Docker Hub
        DOCKER_PASS = credentials('docker-password')  // Credenciales de Docker Hub
        COSIGN_KEY = credentials('cosign-key')        // Clave de Cosign
        WORDPRESS_IMAGE = 'wordpress:latest'  // Usamos la imagen más reciente de Docker Hub
    }

    stages {
        stage('Checkout') {
            steps {
                // Clonamos el repositorio de GitHub
                git 'https://github.com/SuperDebian/TFM_Jenkins.git'
            }
        }

        stage('Pull WordPress Image') {
            steps {
                script {
                    // Descargamos la imagen más reciente de WordPress
                    sh "docker pull ${WORDPRESS_IMAGE}"
                }
            }
        }

        stage('Scan with Trivy') {
            steps {
                script {
                    // Escanear la imagen de WordPress con Trivy para vulnerabilidades
                    sh "trivy image ${WORDPRESS_IMAGE}"
                }
            }
        }

        stage('Sign Image with Cosign') {
            steps {
                script {
                    // Firmar la imagen de Docker con Cosign
                    sh "cosign sign --key ${COSIGN_KEY} ${DOCKER_REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    // Iniciar sesión en Docker Hub
                    sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin"

                    // Subir la imagen de WordPress a Docker Hub
                    sh "docker tag ${WORDPRESS_IMAGE} ${DOCKER_REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${DOCKER_REGISTRY}/${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }
    }

    post {
        always {
            // Limpiar imágenes Docker locales para no llenar espacio
            sh 'docker system prune -f'
        }
    }
}
