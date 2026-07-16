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
    }
    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
    }
    stages {
        stage('Checkout') {
            steps {
                // With "Pipeline script from SCM", Jenkins already knows which
                // repo/branch to pull (configured in the job UI) - this just
                // performs the full checkout into the workspace.
                checkout scm
            }
        }

        stage('Build Maven Project') {
            steps {
                sh 'mvn clean verify'
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
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
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
        }
    }
    post {
        success {
            echo "========================================="
            echo "BUILD SUCCESSFUL"
            echo "========================================="
            echo "Docker Image:   ${IMAGE_NAME}:${BUILD_NUMBER}"
            echo "Latest Image:   ${IMAGE_NAME}:latest"
            echo "GitOps repo updated. ArgoCD will sync automatically."
            echo "========================================="
        }
        failure {
            echo "========================================="
            echo "PIPELINE FAILED - check stage logs above"
            echo "========================================="
        }
        always {
            sh "rm -rf ${GITOPS_DIR}"
        }
    }
}
