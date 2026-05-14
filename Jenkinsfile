pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        SERVICES = 'Discovery-Server auth-service course-service lesson-service enrollment-service discussion-service notification-service payment-service'
        DOCKER_HUB_USER = 'sanchitkumarsingh098931'
        // Replace with your actual credentials ID in Jenkins
        DOCKER_CREDENTIALS_ID = 'docker-hub' 
        SONAR_QUBE_ENV = 'sonarqube'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test Services') {
            steps {
                script {
                    env.SERVICES.split().each { service ->
                        dir(service) {
                            if (isUnix()) {
                                sh 'mvn -B clean verify'
                            } else {
                                bat 'mvn -B clean verify'
                            }
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    env.SERVICES.split().each { service ->
                        dir(service) {
                            withSonarQubeEnv(env.SONAR_QUBE_ENV) {
                                sh "mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=edulearn-${service} -Dsonar.projectName=edulearn-${service}"
                            }
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                script {
                    env.SERVICES.split().each { service ->
                        def imageName = "${env.DOCKER_HUB_USER}/${service.toLowerCase()}:${env.GIT_COMMIT[0..11]}"
                        sh "docker build -t ${imageName} ${service}"
                        
                        // Extreme retry logic for unstable networks (bad record MAC)
                        retry(10) {
                            try {
                                withDockerRegistry([credentialsId: env.DOCKER_CREDENTIALS_ID, url: 'https://index.docker.io/v1/']) {
                                    sh "docker push ${imageName}"
                                    sh "docker tag ${imageName} ${env.DOCKER_HUB_USER}/${service.toLowerCase()}:latest"
                                    sh "docker push ${env.DOCKER_HUB_USER}/${service.toLowerCase()}:latest"
                                }
                            } catch (Exception e) {
                                echo "Push failed for ${service}, clearing auth and retrying in 30 seconds... (Error: ${e.getMessage()})"
                                sh "docker logout"
                                sleep 30
                                throw e
                            }
                        }
                    }
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true, allowEmptyArchive: true
            }
        }

        stage('Update Helm Image Tags') {
            steps {
                script {
                    // Use withCredentials to safely handle the GitHub token
                    withCredentials([string(credentialsId: 'github-token', variable: 'GIT_TOKEN')]) {
                        sh """
                            # 1. Clean up old clones
                            rm -rf helm-repo
                            
                            # 2. Clone the Helm repository using the token safely in an extra header to avoid URL issues
                            git -c http.extraHeader="Authorization: Basic \$(echo -n x-access-token:\$GIT_TOKEN | base64)" \
                                clone https://github.com/sanchitkumarsingh098931/jepkins-ci-cd.git helm-repo
                            
                            cd helm-repo
                            
                            # 3. Update the image tags in values.yaml
                            # We use sed to find the service tag and update it
                            # Format expected: tag: "old-tag"
                            IMAGE_TAG="${env.GIT_COMMIT[0..11]}"
                            sed -i "s/tag: .*/tag: \\"\$IMAGE_TAG\\"/" charts/edulearn/values.yaml
                            
                            # 4. Commit and Push
                            git config user.email "jenkins@edulearn.local"
                            git config user.name "Jenkins CI"
                            git add charts/edulearn/values.yaml
                            
                            # Only commit and push if there are changes
                            if ! git diff --cached --quiet; then
                                git commit -m "ci: update image tags to \$IMAGE_TAG [skip ci]"
                                git -c http.extraHeader="Authorization: Basic \$(echo -n x-access-token:\$GIT_TOKEN | base64)" \
                                    push origin main
                            else
                                echo "No changes detected in Helm tags."
                            fi
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs(deleteDirs: true, disableDeferredWipeout: true)
        }
    }
}
