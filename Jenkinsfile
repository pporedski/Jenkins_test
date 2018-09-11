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
        stage('123'){
            parallel{
                steps {
                    echo '1'
                    echo '2'
                    echo '3'
                }
                    stage('456'){
                        steps {
                            echo '4'
                            echo '5'
                            echo '6'
                        }
                    }
                    stage('789'){
                        steps {
                            echo '7'
                            echo '8'
                            echo '9'
                        }
                    }
                }
            }
        }
    }
}