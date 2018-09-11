pipeline {
    agent any
    stages {
        stage('abc'){
            steps {
                echo 'a'
                echo 'b'
                echo 'c'
            }
        }
        stage('def'){
            steps {
                echo 'd'
                echo 'e'
                echo 'f'
            }
        }   
        stage ('123'){
            parallel{
                stage('456'){
                    steps {
                        echo '1'
                        echo '2'
                        echo '3'
                    }
                }
            }
        }
    }
}