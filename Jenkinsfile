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
            
            // On se place dans le répertoire de l'application pour toutes les opérations docker-compose
            dir(APP_DIR) {
                try {
                    sh '''#!/bin/bash
                        set -e
                        echo "Arrêt et suppression des anciens conteneurs et réseaux..."
                        docker-compose down --remove-orphans || true

                        echo "Reconstruction et démarrage de l'application..."
                        docker-compose up --build -d app
                    '''
                    
                    networkName = sh(script: 'echo "$(basename $(pwd))_default"', returnStdout: true).trim()
                    echo "Le réseau Docker Compose identifié est : ${networkName}"

                    // CORRECTION DE L'ERREUR : Utilisation de l'interpolation de chaîne
                    sh """#!/bin/bash
                        set -e
                        APP_SERVICE_NAME="app"
                        APP_INTERNAL_PORT="8083"
                        
                        echo "Attente de la disponibilité de l'application sur le réseau ${networkName}..."
                        for i in {1..20}; do
                          # Utilisation d'une image curl légère et fiable
                          # Le --fail fait échouer curl si le code HTTP n'est pas 2xx
                          if docker run --rm --network="${networkName}" appropriate/curl -s --fail http://\${APP_SERVICE_NAME}:\${APP_INTERNAL_PORT}/actuator/health > /dev/null; then
                              echo "L'application est saine et répond !"
                              exit 0 # Le script a réussi, on sort de la boucle et du script
                          else
                              echo "En attente de l'application... (\$i/20)"
                              sleep 6
                          fi
                        done
                        
                        echo "L'application n'a pas démarré à temps."
                        exit 1 # Fait échouer le script sh, ce qui déclenchera le 'catch'
                    """
                } catch(e) {
                    echo "Erreur lors du démarrage ou de la vérification de l'application."
                    // Le 'finally' s'occupera du nettoyage, pas besoin de le faire ici.
                    throw e // Fait échouer le build
                }

                // Le scan et le nettoyage se font dans le bloc 'finally'
                // pour s'assurer qu'ils s'exécutent même si le 'try' réussit.
                try {
                    // On sort du dir() pour avoir le bon chemin de workspace
                    def workspaceDir = pwd().split('/')[0..-2].join('/') 
                    
                    echo "Retour à la racine du workspace pour le scan : ${workspaceDir}"

                    sh """#!/bin/bash
                        set -e
                        echo "Création du dossier de sortie pour le rapport ZAP..."
                        rm -rf "${workspaceDir}/${ZAP_REPORT_DIR}"
                        mkdir -p "${workspaceDir}/${ZAP_REPORT_DIR}"

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
                    echo "Nettoyage final : arrêt des conteneurs..."
                    // Le `dir` est implicite car nous sommes toujours dans son scope
                    sh "docker-compose down --remove-orphans"
                }
            } // Fin du bloc dir(APP_DIR)
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
