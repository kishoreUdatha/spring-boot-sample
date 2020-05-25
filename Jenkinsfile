pipeline {
    agent any
    stages {
        stage ('Build') {
            steps {
                bat'mvn clean compile' 
            }
            post {
                success {
                    junit 'target/surefire-reports/**/*.xml' 
                }
            }
        }
    }
}
