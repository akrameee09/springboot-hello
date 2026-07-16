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
        DEPLOYMENT_NAME = 'springboot-hello'
        K8S_NAMESPACE   = 'default'
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
                failure { script { env.FAILED_STAGE = 'Checkout' } }
            }
        }

        stage('Build Maven Project') {
            steps {
                sh 'mvn clean verify'
            }
            post {
                failure { script { env.FAILED_STAGE = 'Build Maven Project' } }
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
                failure { script { env.FAILED_STAGE = 'SonarQube Analysis' } }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
            post {
                failure { script { env.FAILED_STAGE = 'Quality Gate' } }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
                }
            }
            post {
                failure { script { env.FAILED_STAGE = 'Build Docker Image' } }
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
                failure { script { env.FAILED_STAGE = 'Push Docker Image' } }
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
                failure { script { env.FAILED_STAGE = 'Update GitOps Manifest' } }
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGOCD_TOKEN')]) {
                    sh """
                        curl -sk -X POST \
                          -H "Authorization: Bearer \${ARGOCD_TOKEN}" \
                          "https://172.20.10.223:31996/api/v1/applications/${DEPLOYMENT_NAME}/sync"
                    """
                }
            }
            post {
                failure { script { env.FAILED_STAGE = 'Trigger ArgoCD Sync' } }
            }
        }

        stage('Verify Rollout') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    script {
                        // Waits for ArgoCD to sync AND for the new pods to become healthy.
                        // Timeout accounts for ArgoCD's sync delay + pod startup + probes.
                        sh "kubectl rollout status deployment/${DEPLOYMENT_NAME} -n ${K8S_NAMESPACE} --timeout=150s"

                        env.POD_STATUS = sh(
                            script: "kubectl get pods -n ${K8S_NAMESPACE} -l app=${DEPLOYMENT_NAME} -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IMAGE:.spec.containers[0].image,READY:.status.containerStatuses[0].ready",
                            returnStdout: true
                        ).trim()
                    }
                }
            }
            post {
                failure { script { env.FAILED_STAGE = 'Verify Rollout (pods did not become healthy)' } }
            }
        }
    }
    post {
        success {
            emailext (
                to: "${NOTIFY_EMAIL}",
                subject: "✅ Deploy SUCCESS - springboot-hello #${BUILD_NUMBER} - pods healthy",
                body: """
                    <h3>Deployment confirmed healthy in Kubernetes</h3>
                    <p><b>Build:</b> #${BUILD_NUMBER}</p>
                    <p><b>Image:</b> ${IMAGE_NAME}:${BUILD_NUMBER}</p>
                    <p><b>Namespace:</b> ${K8S_NAMESPACE}</p>
                    <p><b>Pod status:</b></p>
                    <pre>${env.POD_STATUS}</pre>
                    <p><b>Console log:</b> <a href="${BUILD_URL}console">${BUILD_URL}console</a></p>
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
