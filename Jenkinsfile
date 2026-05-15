pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    parameters {
        booleanParam(name: 'RUN_SMOKE_TESTS', defaultValue: true, description: 'Run tests named *Smoke* after the full Maven verification stage.')
    }

    environment {
        SERVICES = 'Discovery-Server auth-service course-service lesson-service enrollment-service discussion-service notification-service payment-service'
        MAVEN_OPTS = '-Dmaven.test.failure.ignore=false'
        DOCKERHUB_NAMESPACE = 'sanchitkumarsingh098931'
        DOCKERHUB_CREDENTIALS_ID = 'docker-hub'
        GITHUB_CREDENTIALS_ID = 'github-token'
        SONARQUBE_ENV = 'sonarqube'
        HELM_VALUES_FILE = 'charts/edulearn/values.yaml'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                    echo "Build Image Tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Network Diagnostics') {
            steps {
                script {
                    echo "Checking network MTU and connectivity..."
                    if (isUnix()) {
                        sh "ip addr | grep mtu || ifconfig | grep mtu || true"
                        sh "ping -c 3 google.com || true"
                    } else {
                        bat "netsh interface ipv4 show interfaces"
                        bat "ping google.com"
                    }
                }
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

        stage('Smoke Tests') {
            when {
                expression { return params.RUN_SMOKE_TESTS }
            }
            steps {
                script {
                    env.SERVICES.split().each { service ->
                        dir(service) {
                            if (isUnix()) {
                                sh 'mvn -B -Dtest="*Smoke*" -DfailIfNoTests=false -Dsurefire.failIfNoSpecifiedTests=false test'
                            } else {
                                bat 'mvn -B -Dtest="*Smoke*" -DfailIfNoTests=false -Dsurefire.failIfNoSpecifiedTests=false test'
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
                            withSonarQubeEnv(env.SONARQUBE_ENV) {
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
                    withDockerRegistry([credentialsId: env.DOCKERHUB_CREDENTIALS_ID, url: 'https://index.docker.io/v1/']) {
                        env.SERVICES.split().each { service ->
                            def baseImageName = "${env.DOCKERHUB_NAMESPACE}/${service.toLowerCase()}"
                            def imageName = "${baseImageName}:${env.IMAGE_TAG}"
                            
                            echo "Building and pushing: ${imageName}"
                            sh "docker build -t ${imageName} ${service}"
                            
                            retry(5) {
                                try {
                                    sh "docker push ${imageName}"
                                    sh "docker tag ${imageName} ${baseImageName}:latest"
                                    sh "docker push ${baseImageName}:latest"
                                } catch (Exception e) {
                                    echo "Push failed for ${service}. Waiting 30s before retry..."
                                    sh "docker logout || true"
                                    sleep 30
                                    throw e
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Update Helm Image Tags') {
            steps {
                script {
                    // Use withCredentials to safely handle the GitHub token
                    // We try both string and usernamePassword types for compatibility
                    try {
                        withCredentials([string(credentialsId: env.GITHUB_CREDENTIALS_ID, variable: 'GIT_TOKEN')]) {
                            updateHelmTags(env.GIT_TOKEN)
                        }
                    } catch (Exception e) {
                        echo "Failed with string credential, trying usernamePassword..."
                        withCredentials([usernamePassword(credentialsId: env.GITHUB_CREDENTIALS_ID, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                            updateHelmTags(env.GIT_TOKEN)
                        }
                    }
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true, allowEmptyArchive: true
                archiveArtifacts artifacts: '**/target/surefire-reports/TEST-*.xml', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            cleanWs(deleteDirs: true, disableDeferredWipeout: true)
        }
    }
}

def updateHelmTags(token) {
    sh """
        set -eu
        # 1. Clean up and clone
        rm -rf helm-repo
        git clone https://x-access-token:${token}@github.com/sanchitkumarsingh098931/jepkins-ci-cd.git helm-repo
        
        cd helm-repo
        
        # 2. Update tags
        IMAGE_TAG="${env.IMAGE_TAG}"
        sed -i "s/tag: .*/tag: \\"\$IMAGE_TAG\\"/" charts/edulearn/values.yaml
        
        # 3. Commit and Push
        git config user.email "jenkins@local"
        git config user.name "Jenkins CI"
        git add charts/edulearn/values.yaml
        
        if ! git diff --cached --quiet; then
            git commit -m "ci: update image tags to \$IMAGE_TAG [skip ci]"
            git push https://x-access-token:${token}@github.com/sanchitkumarsingh098931/jepkins-ci-cd.git HEAD:main
        else
            echo "No changes detected in Helm tags."
        fi
    """
}
