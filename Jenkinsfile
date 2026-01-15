pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub_creds')
        IMAGE_REPO = 'mohdayazz/multibranch-flask-app'
        GIT_CREDS = credentials('github_creds')
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Set Immutable Image Tag') {
            steps {
                script {
                    // Git commit SHA = immutable, production-safe
                    IMAGE_TAG = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    IMAGE_NAME = "${IMAGE_REPO}:${IMAGE_TAG}"

                    echo "🚀 Image to be built: ${IMAGE_NAME}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                  docker build -t ${IMAGE_NAME} .
                """
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                withDockerRegistry(
                    credentialsId: 'dockerhub_creds',
                    url: 'https://index.docker.io/v1/'
                ) {
                    sh """
                      docker push ${IMAGE_NAME}
                    """
                }
            }
        }

        stage('Update Kubernetes Manifest (GitOps)') {
            steps {
                sh """
                  sed -i 's|image:.*|image: ${IMAGE_NAME}|' k8s/deployment.yaml
                """
            }
        }

        stage('Commit & Push Manifest (Trigger Argo CD)') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh """
                      git config user.name "mohdayaz06"
                      git config user.email "mohammedayaz.r@gmail.com"

                      git checkout -B main

                      git add k8s/deployment.yaml
                      git commit -m "Deploy ${IMAGE_NAME}" || echo "No changes"

                      git push https://${GIT_CREDS_USR}:${GIT_CREDS_PSW}@github.com/mohdayaz06/Multi-Branch-Prod main
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build complete. Argo CD will deploy ${IMAGE_NAME}"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
