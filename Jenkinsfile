pipeline {
    agent any

    environment {
        GCP_PROJECT_ID         = "sanbox-aldo-prod"
        GCP_REGION             = "us-central1"
        ARTIFACT_REGISTRY_REPO = "container-repository-gemini-at"
        CLOUD_RUN_SERVICE_NAME = "gemini-angular-app"
        ENVIRONMENT_NAME       = "dev"

        GIT_COMMIT_SHORT = sh(
            script: "git rev-parse --short HEAD",
            returnStdout: true
        ).trim()

        IMAGE_TAG = "${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${ARTIFACT_REGISTRY_REPO}/${CLOUD_RUN_SERVICE_NAME}:${GIT_COMMIT_SHORT}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('GCP & Docker Auth') {
            steps {
                sh """
                    set -e

                    gcloud config set project ${GCP_PROJECT_ID}

                    gcloud auth configure-docker ${GCP_REGION}-docker.pkg.dev --quiet

                    echo "GCP project:"
                    gcloud config get-value project

                    echo "Active GCP account:"
                    gcloud config get-value account

                    echo "Docker credential helper:"
                    cat ~/.docker/config.json
                """
            }
        }

        stage('Build and Push Image') {
            steps {
                sh """
                    set -e

                    echo "Building image:"
                    echo "${IMAGE_TAG}"

                    docker build -t ${IMAGE_TAG} .

                    echo "Pushing image:"
                    docker push ${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to Cloud Run') {
            steps {
                sh """
                    set -e

                    gcloud run deploy ${CLOUD_RUN_SERVICE_NAME}-${ENVIRONMENT_NAME} \
                        --image ${IMAGE_TAG} \
                        --region ${GCP_REGION} \
                        --platform managed \
                        --allow-unauthenticated \
                        --set-env-vars="ENVIRONMENT_NAME=${ENVIRONMENT_NAME}"
                """
            }
        }
    }

    post {
        always {
            sh "docker rmi ${IMAGE_TAG} || true"
        }
    }
}
