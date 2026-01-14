pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub_creds')
        IMAGE_REPO = 'mohdayazz/multibranch-flask-app'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    // Sanitize branch name for Docker tag
                    def safeBranch = env.BRANCH_NAME?.replaceAll('[^a-zA-Z0-9_.-]', '-')
                    IMAGE_TAG = safeBranch ? safeBranch.toLowerCase() : 'latest'
                    IMAGE_NAME = "${IMAGE_REPO}:${IMAGE_TAG}"
                    echo "✅ Building image: ${IMAGE_NAME}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                      docker build -t ${IMAGE_NAME} .
                    """
                }
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
                script {
                    // Replace the container image in deployment.yaml with the correct tag
                    sh """
                      sed -i 's|image:.*|image: ${IMAGE_NAME}|' k8s/deployment.yaml
                    """
                }
            }
        }

        stage('Commit Updated Manifest') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github_creds', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    script {
                        sh """
                          git config user.name "mohdayaz06"
                          git config user.email "mohammedayaz.r@gmail.com"

                          # Fix detached HEAD in multibranch pipeline
                          git checkout -B main

                          git add k8s/deployment.yaml
                          git commit -m "Update image to ${IMAGE_NAME}" || echo "No changes to commit"

                          # Push using GitHub credentials
                          git push https://${GIT_USER}:${GIT_TOKEN}@github.com/mohdayaz06/Multi-Branch-Prod main
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Apply the updated manifest to Kubernetes
                    sh """
                      kubectl apply -f k8s/deployment.yaml
                      kubectl rollout restart deployment movie-app
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline successful for branch: ${BRANCH_NAME}"
        }
        failure {
            echo "❌ Pipeline failed for branch: ${BRANCH_NAME}"
        }
    }
}
