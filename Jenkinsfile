pipeline {
    agent any 

    environment {
        GCP_PROJECT_ID         = "sanbox-aldo-prod"
        GCP_REGION             = "us-central1"
        ARTIFACT_REGISTRY_REPO = "container-repository-gemini-at" 
        CLOUD_RUN_SERVICE_NAME = "gemini-angular-app"
        ENVIRONMENT_NAME       = "dev"
        
        GIT_COMMIT_SHORT       = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        IMAGE_TAG              = "${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${ARTIFACT_REGISTRY_REPO}/${CLOUD_RUN_SERVICE_NAME}:${GIT_COMMIT_SHORT}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('GCP & Docker Auth') {
            agent {
                docker { 
                    image 'google/cloud-sdk:stable'
                    args '-v /var/run/docker.sock:/var/run/docker.sock --user root'
                }
            }
            steps {
                script {
                    sh "gcloud config set project ${env.GCP_PROJECT_ID}"
                    sh "gcloud auth configure-docker ${env.GCP_REGION}-docker.pkg.dev --quiet"
                }
            }
        }

        stage('Build and Push Image') {
            agent any
            steps {
                sh """
                docker build -t ${env.IMAGE_TAG} .
                docker push ${env.IMAGE_TAG}
                """
            }
        }

        stage('Deploy to Cloud Run') {
            agent { 
                docker { 
                    image 'google/cloud-sdk:stable'
                    args '--user root'
                } 
            }
            steps {
                sh """
                gcloud run deploy ${env.CLOUD_RUN_SERVICE_NAME}-${env.ENVIRONMENT_NAME} \
                    --image ${env.IMAGE_TAG} \
                    --region ${GCP_REGION} \
                    --platform managed \
                    --allow-unauthenticated \
                    --set-env-vars="ENVIRONMENT_NAME=${env.ENVIRONMENT_NAME}"
                """
            }
        }
    }

    post {
        always {
            script {
                if (env.IMAGE_TAG) {
                    sh "docker rmi ${env.IMAGE_TAG} || true"
                }
            }
        }
    }
}