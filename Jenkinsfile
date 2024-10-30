pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'nodejs'
    }
    environment {
        SCANNER_HOME = tool 'mysonar'
        DOCKER_IMAGE = "praveen22233/mygameapp:${BUILD_NUMBER}" // Moved here for global access
        REGISTRY_CREDENTIALS = credentials('dockerhub')
    }
    stages {
        stage("Clean") {
            steps {
                cleanWs()
            }
        }
        stage("Code") {
            steps {
                git "https://github.com/Praveenchoudary/Kubernetes-project.git"
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('mysonar') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=game -Dsonar.projectKey=game"
                }
            }
        }
        stage("Quality Gates") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar'
                }
            }
        }
        stage("Install dependencies") {
            steps {
                sh 'npm install'
            }
        }
        stage("OWASP") {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image with tag: ${DOCKER_IMAGE}"
                    sh "docker build -t ${DOCKER_IMAGE} ."
                }
            }
        }
        stage("Trivy File System") {
            steps {
                echo "Scanning the file system for vulnerabilities..."
                sh "trivy fs . > trivyfile.txt"
            }
        }
        stage("Image Scan") {
            steps {
                echo "Scanning Docker image ${DOCKER_IMAGE} for vulnerabilities..."
                sh "trivy image ${DOCKER_IMAGE}"
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    echo "Pushing Docker image to Docker Hub..."
                    def dockerImage = docker.image("${DOCKER_IMAGE}")
                    docker.withRegistry('https://index.docker.io/v1/', "dockerhub") {
                        dockerImage.push()
                    }
                }
            }
        }
        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = "Kubernetes-project"
                GIT_USER_NAME = "Praveenchoudary"
            }
            steps {
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        # Configure Git
                        git config user.email "praveenchoudarychinthala@gmail.com"
                        git config user.name "Praveenchoudary"
                        
                        # Update the deployment file with the new image tag
                        sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" deployment/deployment-svc.yml
                        
                        # Stage and commit the modified file
                        git add deployment/deployment-svc.yml
                        if git diff --cached --quiet; then
                            echo "No changes to commit."
                        else
                            git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                            
                            # Push changes to the GitHub repository
                            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:master
                        fi
                    '''
                }
            }
        }
    }
    post {
        always {
            echo 'Slack Notifications'
            slackSend (
                channel: '#k8s-project',
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} \n Build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
            )
        }
    }
}
