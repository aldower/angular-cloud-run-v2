// Autenticación a GCP vía OIDC / Workload Identity Federation (sin llaves JSON). Ver LAB_GCP.md
def gcpAuth() {
    withCredentials([string(credentialsId: 'gcp-oidc-token', variable: 'OIDC_TOKEN')]) {
        sh '''
            umask 077
            printf '%s' "$OIDC_TOKEN" > "$WORKSPACE/.oidc_token"
            gcloud iam workload-identity-pools create-cred-config \
                "projects/$GCP_PROJECT_NUMBER/locations/global/workloadIdentityPools/$WIF_POOL/providers/$WIF_PROVIDER" \
                --service-account="$WIF_SERVICE_ACCOUNT" \
                --credential-source-file="$WORKSPACE/.oidc_token" \
                --output-file="$WORKSPACE/.gcp_cred.json"
            gcloud auth login --cred-file="$WORKSPACE/.gcp_cred.json" --quiet
            gcloud config set project "$GCP_PROJECT_ID"
        '''
    }
}

pipeline {
    agent any 

    environment {
        GCP_PROJECT_ID         = "sanbox-aldo-prod"
        GCP_PROJECT_NUMBER     = "REEMPLAZAR_PROJECT_NUMBER"
        GCP_REGION             = "us-central1"
        ARTIFACT_REGISTRY_REPO = "container-repository-gemini-at" 
        CLOUD_RUN_SERVICE_NAME = "gemini-angular-app"
        ENVIRONMENT_NAME       = "dev"

        WIF_POOL               = "jenkins-pool"
        WIF_PROVIDER           = "jenkins-provider"
        WIF_SERVICE_ACCOUNT    = "jenkins-deployer@sanbox-aldo-prod.iam.gserviceaccount.com"
        
        GIT_COMMIT_SHORT       = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        IMAGE_TAG              = "${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${ARTIFACT_REGISTRY_REPO}/${CLOUD_RUN_SERVICE_NAME}:${GIT_COMMIT_SHORT}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('GCP Auth (OIDC)') {
            agent {
                docker { 
                    image 'google/cloud-sdk:stable'
                    args '--user root'
                    reuseNode true
                }
            }
            steps {
                script {
                    gcpAuth()
                    // Token temporal para el docker login del host contra Artifact Registry
                    sh '''
                        gcloud auth print-access-token > "$WORKSPACE/.access_token"
                        chmod 666 "$WORKSPACE/.access_token"
                    '''
                }
            }
        }

        stage('Build and Push Image') {
            steps {
                sh """
                docker login -u oauth2accesstoken --password-stdin ${env.GCP_REGION}-docker.pkg.dev < .access_token
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
                    reuseNode true
                } 
            }
            steps {
                script {
                    gcpAuth()
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
    }

    post {
        always {
            sh 'rm -f .oidc_token .gcp_cred.json .access_token || true'
            script {
                if (env.IMAGE_TAG) {
                    sh "docker rmi ${env.IMAGE_TAG} || true"
                    sh "docker logout ${env.GCP_REGION}-docker.pkg.dev || true"
                }
            }
        }
    }
}