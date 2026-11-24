pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build images') {
            steps {
                bat 'docker compose build'
            }
        }

        stage('Deploy') {
            steps {
                bat 'docker compose down'
                bat 'docker compose up -d'
            }
        }

        stage('Verify') {
            steps {
                bat 'docker compose ps'
            }
        }
    }

    post {
        success {
            echo 'Deploiement reussi'
        }
        failure {
            echo 'Le pipeline a echoue'
            bat 'docker compose logs'
        }
    }
}