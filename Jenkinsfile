pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        IMAGE_TAG  = ""                 // dynamically set
        IMAGE_NAME = ""                 // dynamically set
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: "${env.BRANCH_NAME}",
                    credentialsId: 'git-cred',
                    url: 'https://github.com/vickeyys/gameboard.git'
            }
        }

        stage('Set Build Metadata') {
            steps {
                script {
                    // IMAGE_TAG = commit hash (better than build number, unique rehta hai)
                    env.IMAGE_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()

                    // IMAGE_NAME set based on branch
                    env.IMAGE_NAME = (env.BRANCH_NAME == 'main') ? "vickeys/boardgame" : "vickeys/boardgame-dev"

                    echo "Building image: ${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                }
            }
        }

        stage('Compile Code') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
            post {
                success {
                    withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
                        sh """
                        curl -X POST -H 'Content-type: application/json' \
                            --data '{"text":"✅ Unit tests passed for *${env.BRANCH_NAME}* build #${BUILD_NUMBER}"}' \
                            $SLACK_WEBHOOK
                        """
                    }
                }
                failure {
                    withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
                        sh """
                        curl -X POST -H 'Content-type: application/json' \
                            --data '{"text":"❌ Unit tests failed for *${env.BRANCH_NAME}* build #${BUILD_NUMBER}"}' \
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
                archiveArtifacts artifacts: 'trivy-fs-report.html', fingerprint: true

                withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
                    script {
                        def buildUrl  = env.BUILD_URL
                        def reportUrl = "${buildUrl}artifact/trivy-fs-report.html"
                        sh """
                        curl -X POST -H 'Content-type: application/json' \
                            --data '{"text":"📊 Trivy report ready for *${env.BRANCH_NAME}*: <${reportUrl}|Download Report>"}' \
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
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker build -t ${env.IMAGE_NAME}:${env.IMAGE_TAG} .
                            docker push ${env.IMAGE_NAME}:${env.IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Update Manifests in GitOps Repo') {
            steps {
                script {
                    dir('manifest-repo') {
                        git branch: "main",
                            credentialsId: 'git-cred',
                            url: 'https://github.com/vickeyys/boardgame-manifest.git'

                        // Path select based on branch
                        def manifestPath = (env.BRANCH_NAME == 'main') ? "prod/deployment.yaml" : "dev/deployment.yaml"

                        // Update image in manifest
                        sh """
                            sed -i 's|image:.*|image: ${env.IMAGE_NAME}:${env.IMAGE_TAG}|' ${manifestPath}
                            git config user.email "jenkins@ci.com"
                            git config user.name "Jenkins CI"
                            git add ${manifestPath}
                            git commit -m "Update image to ${env.IMAGE_NAME}:${env.IMAGE_TAG} for ${env.BRANCH_NAME}"
                            git push origin main
                        """
                    }
                }
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                script {
                    build job: 'boardgame-CD',
                        parameters: [
                            string(name: 'IMAGE_NAME', value: "${env.IMAGE_NAME}"),
                            string(name: 'IMAGE_TAG', value: "${env.IMAGE_TAG}")
                        ],
                        wait: false
                }
            }
        }
    }
}
