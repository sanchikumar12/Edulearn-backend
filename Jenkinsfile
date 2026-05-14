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
                    withDockerRegistry([credentialsId: env.DOCKER_CREDENTIALS_ID, url: 'https://index.docker.io/v1/']) {
                        env.SERVICES.split().each { service ->
                            def imageName = "${env.DOCKER_HUB_USER}/${service.toLowerCase()}:${env.GIT_COMMIT[0..11]}"
                            sh "docker build -t ${imageName} ${service}"
                            
                            // Increased retries and sleep time to handle persistent 'tls: bad record MAC' issues
                            retry(5) {
                                try {
                                    sh "docker push ${imageName}"
                                } catch (Exception e) {
                                    echo "Push failed for ${service}, retrying in 20 seconds... (Error: ${e.getMessage()})"
                                    sleep 20
                                    throw e
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true, allowEmptyArchive: true
                junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
            }
        }
    }

    post {
        always {
            cleanWs(deleteDirs: true, disableDeferredWipeout: true)
        }
    }
}
