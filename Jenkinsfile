pipeline {
    agent any
    stages {
        stage('abc'){
            steps {
                step {
                    echo 'a'
                    echo 'b'
                    echo 'c'
                }
                step {
                    echo 'A'
                    echo 'B'
                    echo 'C'
                }
            }
        }
        stage('def'){
            steps {
                step {
                    echo 'd'
                    echo 'e'
                    echo 'f'
                }
                step {
                    echo 'D'
                    echo 'E'
                    echo 'F'
                }
            }
        }
    }
}