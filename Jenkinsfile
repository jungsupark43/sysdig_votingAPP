// Jenkinsfile for a Docker Compose application
pipeline {
    agent any

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Checking out the Voting App source code...'
                checkout scm
            }
        }

        stage('Build All Services') {
            steps {
                echo 'Building all application services using Docker Compose...'
                // This command reads the 'docker-compose.yml' file and builds all the images
                sh 'docker-compose build'
            }
        }

        stage('Deploy Application Stack') {
            steps {
                echo 'Stopping any old services and deploying the new stack...'
                // This stops and removes all containers defined in the compose file
                sh 'docker-compose down'
                // This creates and starts all containers, networks, and volumes in the background
                sh 'docker-compose up -d'
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
            sh 'docker image prune -f' // Clean up unused images
        }
    }
}
