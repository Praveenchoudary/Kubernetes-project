pipeline { 
    agent any 
    tools { 
        jdk 'jdk17' 
        nodejs 'nodejs' 
    } 
    environment { 
        SCANNER_HOME = tool 'mysonar' 
    } 
    stages { 
        stage ("Clean") { 
            steps { 
                cleanWs() 
            } 
        } 
        stage ("Code") { 
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
        stage ("Quality Gates") { 
            steps { 
                script { 
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar' 
                } 
            } 
        } 
        stage ("Install dependencies") { 
            steps { 
                sh 'npm install' 
            } 
        } 
        stage('Build and Push Docker Image') {
            environment {
                DOCKER_IMAGE = "praveen22233/mygameapp:${BUILD_NUMBER}"
                REGISTRY_CREDENTIALS = credentials('dockerhub')
            }
            steps {
                script {
                    echo "Building Docker image with tag: ${DOCKER_IMAGE}"
                    sh 'docker build -t ${DOCKER_IMAGE} .'
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
                        sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" deployment-service.yml
                        
                        # Stage and commit the modified file
                        git add deployment-service.yml
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
}
