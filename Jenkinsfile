pipeline {
    agent any

    environment {
        // Access environment variables using Jenkins credentials
        DOCKER_REGISTRY = credentials('mom-core-docker-registry')
        GKE_CLUSTER = credentials('mom-core-gke-cluster')
        GKE_ZONE = credentials('gke-zone')
        GCP_PROJECT = credentials('gcp-project')
        GOOGLE_APPLICATION_CREDENTIALS = credentials('google-application-credentials')
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    // Determine environment based on branch
                    def branch = env.GIT_BRANCH
                    
                    if (branch == "origin/main") {
                        env.ENVIRONMENT = 'production'
                    } else if (branch == "origin/staging") {
                        env.ENVIRONMENT = 'staging'
                    } else {
                        error("Unknown branch: ${branch}. This pipeline only supports main and staging branches.")
                    }
                }
                checkout scm
            }
        }

        stage('Configure Docker Authentication') {
            steps {
                script {
                    // Extract the registry's hostname for authentication
                    def registryHost = env.DOCKER_REGISTRY.tokenize('/')[0]
                    sh """
                        sudo gcloud auth configure-docker ${registryHost}
                    """
                }
            }
        }


        stage('Build and Push Docker Image') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'mom-core-docker-registry', variable: 'DOCKER_REGISTRY'), 
                                     string(credentialsId: 'mom-core-gke-cluster', variable: 'GKE_CLUSTER'),
                                     string(credentialsId: 'gke-zone', variable: 'GKE_ZONE')]) {

                        def imageTag = "latest"

                        // Use the environment variable without exposing it in the logs
                        sh "docker build -t ${DOCKER_REGISTRY}/momentum-core:${imageTag} ."

                        // Use masked DOCKER_REGISTRY variable for the Docker push
                        sh "docker push ${DOCKER_REGISTRY}/momentum-core:${imageTag}"
                    }
                }
            }
        }

        // stage('Build Docker Image') {
        //     steps {
        //         script {
        //             // Use latest tag for simplicity
        //             def imageTag = "latest"
        //             sh "sudo docker build -t ${DOCKER_REGISTRY}/momentum-core:${imageTag} ."
        //         }
        //     }
        // }

        // stage('Push Docker Image') {
        //     steps {
        //         script {
        //             // Push the Docker image
        //             def imageTag = "latest"
        //             sh "sudo docker push ${DOCKER_REGISTRY}/momentum-core:${imageTag}"
        //         }
        //     }
        // }

        stage('Configure GKE Authentication') {
            steps {
                script {
                    // Use the service account path from credentials
                    sh """
                    sudo gcloud auth activate-service-account --key-file=${GOOGLE_APPLICATION_CREDENTIALS}
                    sudo gcloud container clusters get-credentials ${GKE_CLUSTER} --zone ${GKE_ZONE} --project ${GCP_PROJECT}
                    """
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                script {
                    def imageTag = "latest"
                    sh """
                    kubectl set image deployment/momentum-core-deployment momentum-core=${DOCKER_REGISTRY}/momentum-core:${imageTag} -n app
                    kubectl rollout status deployment/momentum-core-deployment -n app
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished"

            // Optional cleanup action
            script {
                // Clean up local Docker images
                def imageTag = "latest"
                sh """
                sudo docker rmi ${DOCKER_REGISTRY}/momentum-core:${imageTag} || true
                """
                
                // Perform additional cleanup if necessary
                // For example, delete temporary files, logs, etc.
            }
        }
    }
}
