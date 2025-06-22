pipeline {
    agent any

    environment {
        SONARQUBE = 'SonarQube'
        SONARQUBE_AUTH_TOKEN = credentials('sonar-token')
        DEPENDENCY_CHECK = 'owasp/dependency-check'
        GITHUB_REPO = 'https://github.com/ELHACHLAF/MyAppJava2.git'
        APP_DIR = 'spring-boot-template'
    }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    triggers {
        githubPush()
    }

    stages {
         stage('Checkout') {
            steps {
                cleanWs()
                git url: "${GITHUB_REPO}", branch: 'main',credentialsId: 'jenkins-token'
                sh '''#!/bin/bash
                ls -la
                chmod +x spring-boot-template/mvnw
                '''
            }
        }

        stage('Build') {
            steps {
                dir('spring-boot-template') {
                    sh '''
                     pwd
                    ls -la
                    mvn clean install -DskipTests
                    '''
                }
            }
        }

        stage('Analyse SAST avec SonarQube') {
            steps {
                dir('spring-boot-template') {
                    sh 'mvn clean install -DskipTests sonar:sonar -Dsonar.projectKey=myprojectJava2 -Dsonar.host.url=http://sonarqube:9000 -Dsonar.login=$SONARQUBE_AUTH_TOKEN'
                    }
            }
        }

        
        stage('DAST - ZAP Scan') {
            steps {
                dir('spring-boot-template') {
                sh '''
                    echo "Compilation Maven du projet..."
                    mvn clean package -DskipTests

                    echo "Arrêt et suppression du conteneur app uniquement (sans toucher à postgres)..."
                    docker-compose stop app || true
                    docker-compose rm -f app || true

                    echo "Reconstruction de l'image de l'application (app)..."
                    docker-compose build --no-cache app

                    echo "Démarrage du service app (postgres doit déjà être up)..."
                    docker-compose up -d app

                    echo "Vérification que l'application est disponible..."
                    for i in {1..10}; do
                        if curl -s http://host.docker.internal:8081 > /dev/null; then
                            echo "Application disponible !"
                        break
                        else
                            echo "En attente de l'application..."
                            sleep 10
                        fi
                        done
                    # Vérifier que l'image ZAP est disponible
                    docker pull ghcr.io/zaproxy/zaproxy:latest
                    echo "Lancement du scan DAST avec ZAP..."
                '''
                script {
                    try{
                        sh '''
                            docker run --rm -v "$(pwd)/spring-boot-template:/zap/wrk/" \
                                ghcr.io/zaproxy/zaproxy:latest zap-baseline.py \
                                -t http://host.docker.internal:8081 \
                                -r zap_report.html
                                echo "Contenu du dossier après scan :"
                                ls -l
                        '''
                    }
                    catch (err) {
                      echo "ZAP scan a détecté des vulnérabilités critiques (code de sortie non nul)."
                      // continuer malgré l’erreur pour archiver le rapport
                    }
                }

            
        }
    }
}
    }
    post {
    always {
      archiveArtifacts artifacts: 'spring-boot-template/zap_report.html', allowEmptyArchive: true
      publishHTML(target: [
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'spring-boot-template',
                    reportFiles: 'zap_report.html',
                    reportName: 'ZAP Report'
                ])
    }
}

}
