pipeline {
    agent any

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch to deploy')
    }

    environment {
        JENKINS_JOB_WEBHOOK_URL = credentials('jenkins-job-webhook-url')
        // Set COMPOSE_PROJECT_NAME to avoid conflicts in other compose projects run nearby
        COMPOSE_PROJECT_NAME = 'docker-app-ai'
        // Define the compose file path relative to the workspace root
        COMPOSE_FILE = 'docker-compose.yaml'
    }

    options {
        // Prevent the first automatic checkout
        skipDefaultCheckout()
    }

    stages {
        stage('Checkout') {
            steps {
                deleteDir() // clean start to avoid partial git dirs
                checkout scm
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                echo "Deploying AI Stack using ${env.COMPOSE_FILE}..."
                withCredentials([
                    file(credentialsId: 'ai-env-file', variable: 'ENV_FILE')
                ]) {
                    sh """
                        echo "[+] Injecting .env from Jenkins secret"
                        cp "${ENV_FILE}" .env

                        echo "[+] Pulling latest images..."
                        docker compose -f ${env.COMPOSE_FILE} pull

                        echo "[+] Starting AI Stack Services..."
                        # Use -d for detached mode
                        docker compose -f ${env.COMPOSE_FILE} up -d --remove-orphans --force-recreate
                    """
                }
            }
        }

        stage('Post-Deploy Check') {
            steps {
                echo "[+] Verifying running containers for project ${env.COMPOSE_PROJECT_NAME}..."
                // Basic check to see if containers are running
                sh "docker compose -f ${env.COMPOSE_FILE} ps"
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
            // Optional: Clean up workspace if desired.
            cleanWs()
        }
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed.'
            // Send failure notification to pumble
            sh """
                curl -X POST -H 'Content-type: application/json' \\
                --data '{
                    "text": "🚨 Build Failed! Job: `${env.JOB_NAME}` Build: `${env.BUILD_NUMBER}`. Check logs: ${env.BUILD_URL}"
                }' \\
                ${JENKINS_JOB_WEBHOOK_URL}
            """
        }
    }
}