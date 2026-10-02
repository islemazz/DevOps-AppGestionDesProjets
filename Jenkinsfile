pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'islemaz'
        TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build images') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Deploy') {
            steps {
                sh 'docker compose up -d'
            }
        }

        stage('Smoke test') {
            steps {
                sh '''
                    for i in $(seq 1 30); do
                        if curl -sf http://localhost:8083/entreprise/all; then
                            echo "\nApplication OK"; exit 0
                        fi
                        echo "Attente du backend... ($i/30)"; sleep 5
                    done
                    echo "Application KO"; exit 1
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        for app in backend frontend; do
                            docker tag gestion-$app:1.0 $DOCKERHUB_USER/gestion-$app:$TAG
                            docker tag gestion-$app:1.0 $DOCKERHUB_USER/gestion-$app:latest
                            docker push $DOCKERHUB_USER/gestion-$app:$TAG
                            docker push $DOCKERHUB_USER/gestion-$app:latest
                        done
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
    }
}
