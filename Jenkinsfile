pipeline {
    agent any

    environment {
        IMAGE_NAME = "tugas2-secure-app"
        IMAGE_TAG  = "${env.BUILD_NUMBER ?: 'latest'}"
    }

    stages {
        stage('Build Docker Image') {
            steps {
                sh 'docker build --network=host -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }

        stage('Trivy Misconfiguration Scan') {
            steps {
                sh 'trivy config --exit-code 1 --severity CRITICAL,HIGH .'
            }
        }

        stage('Trivy Vulnerability Gate') {
            steps {
                sh 'trivy image --exit-code 1 --severity CRITICAL,HIGH ${IMAGE_NAME}:${IMAGE_TAG}'
            }
        }
    }

    post {
        always {
            sh 'docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true'
        }
    }
}
