pipeline {
    environment {
    str = 'Its me Mario'
    }
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
        stage('numbers'){
            parallel{
                stage('123'){
                    steps {
                        echo '1'
                        echo '2'
                        echo '3'
                    }                
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
        stage('Who is there'){
                steps{
                echo "${str}"
            }
        }
        stage('for'){
            steps{
                script{
                    def character = ['Mario','Luigi','Yoshi','Bowser']
                    for (int i = 0; i < character.size(); ++i) {
                        echo "I am ${character[i]}"
                    }
                }
            }
        }
        stage('bla'){
            steps{
                script{
                    def colors = ['blue','red','green','yellow']
                }
            }
        }
    }
    def colorsTest(colors) {
        for(int i = 0; i < colors.size(); ++i) {
            echo "I know these colors: ${colors[i]}"
        }
    }
}
