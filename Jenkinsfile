@Library("Shared") _
pipeline {
    agent { label "vinod" }

    stages {
        stage("Hello"){
            steps{
                script{
                    hello()
                }
            }
        }
        stage("Code") {
            steps {
                script{
                    code_checkout("https://github.com/deck-kilyani/django-notes-app.git", "main")
                }
            }
        }

        stage("Build") {
            steps {
                script{
                    docker_build("notes-app", "latest", "deckkilyani")
                }
            }
        }

        stage("Push To DockerHub") {
            steps {
                script{
                    docker_push("notes-app", "latest", "deckkilyani")
                }
            }
        }

        stage("Test") {
            steps {
                echo "This is testing the code"
            }
        }

        stage("Deploy") {
            steps {
                echo "This is deploying the code"
                sh "docker compose up -d"
            }
        }
    }
}
