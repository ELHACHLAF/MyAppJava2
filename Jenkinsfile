pipeline {
    agent any

    environment {
        SONARQUBE = 'SonarQube'
        SONARQUBE_AUTH_TOKEN = credentials('sonar-token')
        DEPENDENCY_CHECK = 'owasp/dependency-check'
        GITHUB_REPO = 'https://github.com/ELHACHLAF/MyAppJava2.git'
    }

    tools {
        maven 'maven3'
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                git url: "${GITHUB_REPO}", branch: 'main', credentialsId: 'jenkins-token'
                
            }
        }

        stage('Build & Unit Tests') {
            steps {
                dir('spring-boot-template') {
                sh 'chmod +x ./mvnw'
                sh './mvnw clean verify'
            }
        }
        }

        stage('Analyse SAST avec SonarQube') {
            steps {
                withSonarQubeEnv("${SONARQUBE}") {
                    sh """
                        ./mvnw sonar:sonar \
                        -Dsonar.projectKey=spring-boot-template \
                        -Dsonar.host.url=http://sonarqube:9000 \
                        -Dsonar.login=${SONARQUBE_AUTH_TOKEN} \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    """
                }
            }
        }

        stage('Analyse SCA avec Dependency-Check') {
            steps {
                script {
                    sh 'mkdir -p reports'
                    sh """
                        docker run --rm -m 2g \
                            -v "\$WORKSPACE:/src" \
                            -v "\$WORKSPACE/reports:/report" \
                            ${DEPENDENCY_CHECK} \
                            --project 'spring-boot-template' \
                            --scan /src \
                            --format HTML \
                            --out /report \
                            --nvdApiKey 7190945a-3597-41eb-9558-c0879ee2efcd
                    """
                    archiveArtifacts artifacts: 'reports/*.html', allowEmptyArchive: true
                }
            }
        }
    }

}
