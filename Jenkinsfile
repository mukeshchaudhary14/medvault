pipeline {
    agent any

    environment {
        // Docker Hub details
        DOCKER_HUB_USER    = 'makug'
        DOCKER_IMAGE_NAME  = 'medvault'
        DOCKER_REPOSITORY  = "${DOCKER_HUB_USER}/${DOCKER_IMAGE_NAME}"
        
        // Dynamic build tag
        IMAGE_TAG          = "${env.BUILD_NUMBER}"
        
        // Jenkins Credentials ID configured in Jenkins (Manage Jenkins -> Credentials)
        DOCKER_CREDENTIALS = 'docker-hub-credentials'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10')) // Purge old builds to save disk space
        timeout(time: 30, unit: 'MINUTES')             // Kill hanging jobs
        timestamps()                                   // Add timestamps in console logs
    }

    stages {

        // =====================================================================
        // STAGE 1: Code Checkout
        // =====================================================================
        stage('Checkout Source Code') {
            steps {
                echo "📥 Checking out source code from repository..."
                checkout scm
            }
        }

        // =====================================================================
        // STAGE 2: Code Quality & Build Verification
        // =====================================================================
        stage('Code Quality & Test') {
            steps {
                echo "🧪 Running lint and checking TypeScript build..."
                sh '''
                    # Agar machine me nodejs hai toh direct run hoga,
                    # nahi toh Docker container ke andar isolated test chalega:
                    if command -v node >/dev/null 2>&1; then
                        npm ci || npm install --legacy-peer-deps
                        npm run lint || true
                        npm run build
                    else
                        docker run --rm -v "$(pwd)":/app -w /app node:22-alpine sh -c "
                            npm ci || npm install --legacy-peer-deps
                            npm run lint || true
                            npm run build
                        "
                    fi
                '''
            }
        }

        // =====================================================================
        // STAGE 3: Build Docker Image
        // =====================================================================
        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker Image: ${DOCKER_REPOSITORY}:${IMAGE_TAG} and latest..."
                sh """
                    docker build -t ${DOCKER_REPOSITORY}:${IMAGE_TAG} -t ${DOCKER_REPOSITORY}:latest .
                """
            }
        }

        // =====================================================================
        // STAGE 4: Security Scan with Trivy
        // =====================================================================
        stage('Security Scan (Trivy)') {
            steps {
                echo "🛡️ Scanning Docker Image for Critical/High vulnerabilities..."
                sh """
                    if command -v trivy >/dev/null 2>&1; then
                        trivy image --severity HIGH,CRITICAL --format table ${DOCKER_REPOSITORY}:${IMAGE_TAG} || true
                    else
                        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy:latest image --severity HIGH,CRITICAL ${DOCKER_REPOSITORY}:${IMAGE_TAG} || true
                    fi
                """
            }
        }

        // =====================================================================
        // STAGE 5: Push Image to Docker Hub
        // =====================================================================
        stage('Push to Docker Hub') {
            steps {
                echo "🚀 Pushing image to Docker Hub under ${DOCKER_REPOSITORY}..."
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS}", usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh """
                        echo "\$DH_PASS" | docker login -u "\$DH_USER" --password-stdin
                        docker push ${DOCKER_REPOSITORY}:${IMAGE_TAG}
                        docker push ${DOCKER_REPOSITORY}:latest
                    """
                }
            }
        }

        // =====================================================================
        // STAGE 6: Deploy from Docker Hub
        // =====================================================================
        stage('Deploy (Pull from Docker Hub)') {
            steps {
                echo "🌐 Pulling and deploying container directly from Docker Hub..."
                sh """
                    # Pull fresh image from Docker Hub (No source code needed on deploy target)
                    docker pull ${DOCKER_REPOSITORY}:latest

                    # Stop and remove older container if exists
                    docker stop medvault-app || true
                    docker rm medvault-app || true

                    # Run new container from freshly pulled image
                    docker run -d \
                        --name medvault-app \
                        --restart unless-stopped \
                        -p 5173:80 \
                        ${DOCKER_REPOSITORY}:latest
                """
            }
        }
    }

    // =========================================================================
    // POST BUILD ACTIONS & CLEANUP
    // =========================================================================
    post {
        always {
            echo "🧹 Cleaning up workspace & dangling images..."
            sh '''
                docker logout || true
                docker image prune -f || true
            '''
            cleanWs()
        }
        success {
            echo "✅ ========================================================"
            echo "✅ MedVault Pipeline Completed Successfully!"
            echo "✅ Image: ${DOCKER_REPOSITORY}:${IMAGE_TAG} pushed to Docker Hub."
            echo "✅ App is Live at: http://<SERVER_IP>:5173"
            echo "✅ ========================================================"
        }
        failure {
            echo "❌ Pipeline execution failed. Please inspect console logs."
        }
    }
}
