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
/*
        stage('Analyse SAST avec SonarQube') {
            steps {
                dir('spring-boot-template') {
                    sh 'mvn clean install -DskipTests sonar:sonar -Dsonar.projectKey=myprojectJava2 -Dsonar.host.url=http://sonarqube:9000 -Dsonar.login=$SONARQUBE_AUTH_TOKEN'
                    }
            }
        }

      */  
        stage('DAST - ZAP Scan') {
    steps {
        script {
            def networkName = ''
            
            // Étape 1 : Démarrer l'application avec docker-compose
            // On utilise dir() uniquement pour ce bloc
            dir(APP_DIR) {
                try {
                    sh '''#!/bin/bash
                        set -e
                        echo "Arrêt et suppression des anciens conteneurs et réseaux..."
                        docker-compose down --remove-orphans || true

                        echo "Reconstruction et démarrage de l'application..."
                        docker-compose up --build -d app
                    '''
                    
                    // On récupère le nom du réseau pendant qu'on est dans le bon dossier
                    networkName = sh(script: 'echo "$(basename $(pwd))_default"', returnStdout: true).trim()
                    echo "Le réseau Docker Compose identifié est : ${networkName}"

                    // On vérifie que l'app est prête
                    sh '''#!/bin/bash
                        set -e
                        APP_SERVICE_NAME="app"
                        APP_INTERNAL_PORT="8083"
                        DOCKER_NETWORK_NAME=$1 // Récupère le nom du réseau passé en argument

                        echo "Attente de la disponibilité de l'application sur le réseau ${DOCKER_NETWORK_NAME}..."
                        for i in {1..20}; do
                          if docker run --rm --network=${DOCKER_NETWORK_NAME} appropriate/curl -s --fail http://${APP_SERVICE_NAME}:${APP_INTERNAL_PORT}/actuator/health > /dev/null; then
                              echo "L'application est saine et répond !"
                              break
                          else
                              echo "En attente de l'application... ($i/20)"
                              sleep 6
                          fi
                        done
                    '''.execute(networkName) // Passe la variable Groovy au script shell
                    
                } catch(e) {
                    echo "Erreur lors du démarrage de l'application avec docker-compose."
                    // On s'assure de nettoyer même en cas d'échec
                    dir(APP_DIR) {
                        sh "docker-compose down --remove-orphans"
                    }
                    throw e // Fait échouer le build
                }
            } // Fin du bloc dir(APP_DIR)

            // Maintenant, le répertoire de travail courant est de nouveau la racine du workspace.
            echo "Retour à la racine du workspace : ${pwd()}"

            // Étape 2 : Lancer le scan ZAP depuis la racine du workspace
            if (networkName) { // On ne lance le scan que si le réseau a été créé
                try {
                    def workspaceDir = pwd() // Maintenant, pwd() donne le bon chemin
                    
                    sh """#!/bin/bash
                        set -e
                        echo "Création du dossier de sortie pour le rapport ZAP..."
                        rm -rf "${ZAP_REPORT_DIR}"
                        mkdir -p "${ZAP_REPORT_DIR}"

                        echo "Lancement du scan ZAP sur le réseau ${networkName}..."
                        
                        docker run --rm --network "${networkName}" \
                            -v "${workspaceDir}":/zap/wrk \
                            ghcr.io/zaproxy/zaproxy:latest \
                            zap-baseline.py \
                            -t http://app:8083 \
                            -r "wrk/${ZAP_REPORT_DIR}/zap_report.html" \
                            -d
                    """
                } catch (err) {
                    echo "⚠️ ZAP a terminé avec un statut de non-succès. Des vulnérabilités ont probablement été trouvées."
                } finally {
                    // Étape 3 : Nettoyage final
                    echo "Arrêt des conteneurs après le scan..."
                    // On doit retourner dans le dossier pour que docker-compose trouve son fichier
                    dir(APP_DIR) {
                        sh "docker-compose down --remove-orphans"
                    }
                }
            } else {
                error("Le nom du réseau Docker n'a pas pu être déterminé. Le scan ZAP est annulé.")
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
