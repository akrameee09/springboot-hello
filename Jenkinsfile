pipeline {
    agent any
    tools {
        maven 'maven-3.9.16'
    }
    environment {
        SONARQUBE_ENV   = 'Sonar-Server'
        IMAGE_NAME      = 'akrameee09/springboot-hello'
        GITOPS_REPO_URL = 'github.com/akrameee09/springboot-hello-gitops.git'
        GITOPS_DIR      = 'gitops-repo'
        NOTIFY_EMAIL    = 'akramhossain.se@gmail.com'
    }
    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
            post {
                failure {
                    script { env.FAILED_STAGE = 'Checkout' }
                }
            }
        }

        stage('Build Maven Project') {
            steps {
                sh 'mvn clean verify'
            }
            post {
                failure {
                    script { env.FAILED_STAGE = 'Build Maven Project' }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                    mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988:sonar \
                    -Dsonar.projectKey=springboot-hello \
                    -Dsonar.projectName=springboot-hello
                    '''
                }
            }
            post {
                failure {
                    script { env.FAILED_STAGE = 'SonarQube Analysis' }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
            post {
                failure {
                    script { env.FAILED_STAGE = 'Quality Gate' }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
                }
            }
            post {
                failure {
                    script { env.FAILED_STAGE = 'Build Docker Image' }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') {
                        dockerImage.push("${BUILD_NUMBER}")
                        dockerImage.push("latest")
                    }
                }
            }
            post {
                failure {
                    script { env.FAILED_STAGE = 'Push Docker Image' }
                }
            }
        }

        stage('Update GitOps Manifest') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-cred',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh """
                        rm -rf ${GITOPS_DIR}
                        git clone https://\${GIT_USER}:\${GIT_TOKEN}@${GITOPS_REPO_URL} ${GITOPS_DIR}
                        cd ${GITOPS_DIR}

                        sed -i 's|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${BUILD_NUMBER}|' k8s/deployment.yaml

                        git config user.name "Jenkins"
                        git config user.email "jenkins@local"
                        git add k8s/deployment.yaml
                        git diff --cached --quiet && echo "No changes to commit" || (
                            git commit -m "Deploy build ${BUILD_NUMBER}"
                            git push https://\${GIT_USER}:\${GIT_TOKEN}@${GITOPS_REPO_URL} HEAD:main
                        )
                    """
                }
            }
            post {
                failure {
                    script { env.FAILED_STAGE = 'Update GitOps Manifest' }
                }
            }
        }
    }
    post {
        success {
            emailext (
                to: "${NOTIFY_EMAIL}",
                subject: "✅ Deploy SUCCESS - springboot-hello #${BUILD_NUMBER}",
                body: """
                    <h3>Deployment pipeline completed successfully</h3>
                    <p><b>Build:</b> #${BUILD_NUMBER}</p>
                    <p><b>Image:</b> ${IMAGE_NAME}:${BUILD_NUMBER}</p>
                    <p><b>GitOps repo:</b> updated - ArgoCD will sync automatically</p>
                    <p><b>Console log:</b> <a href="${BUILD_URL}console">${BUILD_URL}console</a></p>
                    <p style="color:#888;font-size:12px">Note: this confirms the build/push/manifest-update succeeded.
                    Actual pod health in the cluster is managed by ArgoCD - check the ArgoCD UI to confirm Synced/Healthy.</p>
                """,
                mimeType: 'text/html'
            )
        }
        failure {
            emailext (
                to: "${NOTIFY_EMAIL}",
                subject: "❌ Deploy FAILED - springboot-hello #${BUILD_NUMBER} - Stage: ${env.FAILED_STAGE ?: 'Unknown'}",
                body: """
                    <h3>Deployment pipeline failed</h3>
                    <p><b>Build:</b> #${BUILD_NUMBER}</p>
                    <p><b>Failed stage:</b> ${env.FAILED_STAGE ?: 'Unknown'}</p>
                    <p><b>Console log:</b> <a href="${BUILD_URL}console">${BUILD_URL}console</a></p>
                """,
                mimeType: 'text/html'
            )
        }
        always {
            sh "rm -rf ${GITOPS_DIR}"
        }
    }
}
