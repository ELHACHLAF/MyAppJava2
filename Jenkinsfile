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

        stage('Analyse SCA avec Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./spring-boot-template --format XML --noupdate', odcInstallation: 'owasp-dependency'
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'

            }
        }
        stage('DAST - ZAP Scan') {
            steps {
                dir('spring-boot-template') {
                // Lancer docker-compose
                sh '''
                docker-compose up -d
                echo "Attente du démarrage de l'application..."
                sleep 30
                mvn spring-boot:run &
                # Attend 30 secondes que l'app soit prête
                sleep 30
                 # Vérifie si le conteneur est bien lancé
                docker-compose ps

                # Optionnel : tester que l'appli répond avant de lancer ZAP (requiert curl dans l'image Jenkins)
                echo "Vérification de la disponibilité de l'application..."
                for i in {1..10}; do
                    if curl -s http://host.docker.internal:8081 > /dev/null; then
                    echo "Application disponible !"
                    break
                else
                    echo "En attente de l'application..."
                    sleep 10
                fi
                done
                '''
            }
            }
        }
    }

}
