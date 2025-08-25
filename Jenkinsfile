
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        IMAGE_NAME = "vickeys/boardgame"
        IMAGE_TAG  = "${BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/vickeyys/Boardgame.git'
            }
        }

        stage('Compile Code') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Unit Test') {
            when {
                expression { env.SKIP_TESTS != 'true' }   // ✅ skip only if SKIP_TESTS=true
            }
            steps {
                sh 'mvn test'
            }
            post {
                success {
                    withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
                        sh """
                        curl -X POST -H 'Content-type: application/json' \
                            --data '{"text":"✅ Unit tests passed successfully for build #${BUILD_NUMBER}"}' \
                            $SLACK_WEBHOOK
                        """
                    }
                }
                failure {
                    withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
                        sh """
                        curl -X POST -H 'Content-type: application/json' \
                            --data '{"text":"❌ Unit tests failed for build #${BUILD_NUMBER}"}' \
                            $SLACK_WEBHOOK
                        """
                    }
                }
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('Upload Artifact & Notify Slack') {
            steps {
                // Archive Trivy scan report
                archiveArtifacts artifacts: 'trivy-fs-report.html', fingerprint: true

                // Notify Slack with artifact link
                withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
                    script {
                        def buildUrl  = env.BUILD_URL
                        def reportUrl = "${buildUrl}artifact/trivy-fs-report.html"
                        sh """
                        curl -X POST -H 'Content-type: application/json' \
                            --data '{"text":"📊 Trivy report is ready: <${reportUrl}|Download Report>"}' \
                            $SLACK_WEBHOOK
                        """
                    }
                }
            }
        }

        stage('Build JAR') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                script {
                    build job: 'boardgame-CD',
                        parameters: [
                            string(name: 'IMAGE_TAG', value: "${IMAGE_TAG}"),
                            string(name: 'IMAGE_NAME', value: "${IMAGE_NAME}")
                        ],
                        wait: false   // don't block CI while CD runs
                }
            }
        }
    }
}
