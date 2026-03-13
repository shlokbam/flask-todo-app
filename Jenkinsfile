pipeline {
    agent any

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com/shlokbam/flask-todo-app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-todo-app:latest .'
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d --build'
            }
        }

        stage('Deployment Status') {
            steps {
                sh 'docker ps'
                echo 'Deployment successful! App running on port 5000.'
            }
        }
    }
}