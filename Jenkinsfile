pipeline {
    agent any

    environment {
        APP_PORT = '5000'
        ZAP_PORT = '8090'
        DOCKER_NET = 'devsecops-lab'
    }

    stages {

        // ── STAGE 1 : Récupérer le code ──
        stage('Checkout') {
            steps {
                echo 'Récupération du code source...'
                checkout scm
            }
        }

        // ── STAGE 2 : Builder l'image CI ──
        stage('Build CI Image') {
            steps {
                echo 'Construction de l image CI...'
                sh 'docker build -f Dockerfile.ci -t devsecops-ci:latest .'
            }
        }

        // ── STAGE 3 : Build et tests unitaires ──
        stage('Build & Test') {
            agent {
                docker {
                    image 'devsecops-ci:latest'
                    reuseNode true
                }
            }
            steps {
                echo 'Exécution des tests unitaires...'
                sh 'pytest tests/ -v'
            }
        }

        // ── STAGE 4 : SAST avec Bandit ──
        stage('SAST - Bandit Security Scan') {
            agent {
                docker {
                    image 'devsecops-ci:latest'
                    reuseNode true
                }
            }
            steps {
                echo 'Analyse de sécurité statique du code (SAST)...'
                sh 'bandit -r app/ -f json -o bandit-report.json || true'
                sh 'bandit -r app/ || true'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'bandit-report.json',
                        allowEmptyArchive: true
                }
            }
        }

        // ── STAGE 5 : Build de l'image Docker de l'app ──
        stage('Docker Build') {
            steps {
                echo 'Construction de l image Docker de l application...'
                sh 'docker build -t devsecops-app:latest .'
            }
        }

        // ── STAGE 6 : DAST avec OWASP ZAP ──
        stage('DAST - OWASP ZAP Pentest') {
            steps {
                echo 'Lancement du pentest dynamique avec OWASP ZAP...'
                sh '''
                    docker run -d \
                        --name target-app \
                        --network ${DOCKER_NET} \
                        -p ${APP_PORT}:5000 \
                        devsecops-app:latest
                    sleep 5
                '''
                sh '''
                    chmod 775 $(pwd)
                '''
                sh '''
                    docker run --rm \
                        --network ${DOCKER_NET} \
                        -v $(pwd):/zap/wrk/:rw \
                        -u root \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t http://target-app:5000 \
                        -r zap-report.html \
                        -J zap-report.json \
                        -I
                    ls -la $(pwd)/zap-report.*
                '''
            }
            post {
                always {
                    sh 'docker stop target-app || true'
                    sh 'docker rm target-app || true'
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'zap-report.html',
                        reportName: 'ZAP Security Report'
                    ])
                    archiveArtifacts artifacts: 'zap-report.json',
                        allowEmptyArchive: true
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline terminé ! Consulte les rapports de sécurité.'
        }
        failure {
            echo 'Pipeline échoué. Regarde les logs pour plus de détails.'
        }
    }
}