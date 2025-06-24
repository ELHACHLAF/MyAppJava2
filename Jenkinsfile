pipeline {
    agent any

    environment {
        // --- Variables de configuration ---
        GITHUB_REPO        = 'https://github.com/ELHACHLAF/MyAppJava2.git'
        APP_DIR            = 'spring-boot-template'
        ZAP_REPORT_DIR     = 'zap-reports' // Dossier de rapport propre
        
        // --- Variables spécifiques à l'application ---
        APP_SERVICE_NAME   = 'app'
        APP_INTERNAL_PORT  = '8080' // **CORRECTION CRITIQUE : C'est le port interne, pas 8083**
        APP_HEALTH_ENDPOINT= '/actuator/health'
        
        // --- Variables pour les autres outils ---
        SONARQUBE          = 'SonarQube'
        SONARQUBE_AUTH_TOKEN = credentials('sonar-token')
    }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    stages {
         stage('Checkout & Build') {
            steps {
                cleanWs()
                git url: "${GITHUB_REPO}", branch: 'main', credentialsId: 'jenkins-token'
                dir(APP_DIR) {
                    // La commande chmod n'est plus nécessaire si on n'utilise pas mvnw
                    // sh 'chmod +x mvnw' 
                    
                    // CORRECTION : On utilise l'outil 'mvn' global de Jenkins
                    sh 'mvn clean install -DskipTests'
                }
            }
        }
        
        
        stage('DAST - ZAP Scan') {
            steps {
                script {
                    dir(APP_DIR) {
                        def networkName = sh(script: 'echo "$(basename $(pwd))_default"', returnStdout: true).trim()
                        
                        try {
                            sh '''#!/bin/bash
                                set -e
                                echo "Arrêt et suppression des anciens conteneurs et volumes..."
                                docker-compose down --volumes --remove-orphans || true

                                echo "Reconstruction et démarrage des services..."
                                docker-compose up --build -d
                            '''
                            
                            echo "Le réseau Docker Compose identifié est : ${networkName}"

                            // Script de vérification de santé. Il est plus fiable grâce aux modifications du code.
                            sh """#!/bin/bash
                                set -e
                                echo "Attente que l'application soit saine (HEALTHY)..."
                                for i in {1..30}; do
                                  # On vérifie directement l'état de santé géré par Docker
                                  status=\$(docker inspect -f '{{.State.Health.Status}}' "${APP_SERVICE_NAME}" 2>/dev/null)
                                  if [ "\$status" == "healthy" ]; then
                                      echo "L'application est saine (HEALTHY) !"
                                      break
                                  else
                                      echo "En attente de l'application... (état: \${status:-"starting"}, essai: \$i/30)"
                                      sleep 6
                                  fi
                                  if [ \$i -eq 30 ]; then
                                    echo "L'application n'est pas devenue saine à temps."
                                    docker-compose logs # Affiche les logs pour le débogage
                                    exit 1
                                  fi
                                done
                            """
                            
                            // --- DÉBUT DU SCAN ZAP ---
                            sh 'docker pull ghcr.io/zaproxy/zaproxy:latest'
                            def workspaceDir = pwd().split('/')[0..-2].join('/') 

                            sh """#!/bin/bash
                                set -e
                                echo "Création du dossier de sortie pour le rapport ZAP..."
                                rm -rf "${workspaceDir}/${ZAP_REPORT_DIR}"
                                mkdir -p "${workspaceDir}/${ZAP_REPORT_DIR}"

                                echo "Lancement du scan ZAP de base sur http://${APP_SERVICE_NAME}:${APP_INTERNAL_PORT}..."
                                
                                docker run --rm --network "${networkName}" \
                                    -v "${workspaceDir}":/zap/wrk \
                                    ghcr.io/zaproxy/zaproxy:latest \
                                    zap-baseline.py \
                                    -t http://${APP_SERVICE_NAME}:${APP_INTERNAL_PORT} \
                                    -r "wrk/${ZAP_REPORT_DIR}/zap_baseline_report.html"
                            """
                        } catch (err) {
                            echo "⚠️ Une erreur est survenue pendant le stage DAST."
                            // Affiche les logs pour aider au débogage
                            sh "docker-compose logs"
                        } finally {
                            echo "Nettoyage final : arrêt des conteneurs..."
                            sh "docker-compose down --volumes --remove-orphans"
                        }
                    } // Fin du bloc dir(APP_DIR)
                }
            }
        }
    }

    post {
        always {
            // CORRECTION : Le chemin doit correspondre au dossier et nom de fichier définis plus haut
            archiveArtifacts artifacts: "${ZAP_REPORT_DIR}/zap_baseline_report.html", allowEmptyArchive: true
            
            publishHTML(target: [
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: ZAP_REPORT_DIR,
                reportFiles: 'zap_baseline_report.html',
                reportName: 'ZAP Baseline Report'
            ])
        }
    }
}
