pipeline {
    environment {
        //Variables need to be defined here
        stringTest = 'This is a test string'
        blub = 'blub'
        didSucceed = true
    }

    agent any
    stages {
        stage('Echo') {
            steps {
                //Simple echo testing
                echo 'One'
                echo 'Two'
            }

        }

        stage('Variable') {
            steps {
                //Testing if you can declare a variable anywhere and use it anywhere
                echo "${stringTest}"
                stringTest()
            }
        }
            
        stage('Tests') {
            parallel {
                //Testing if you can pass items while in a parallel stage
                stage('Pass list'){
                    steps {
                        //Passing lists tests
                        listTest(['a', 'b', 'c'])
                    }
                }

                stage('Parallel') {
                    steps {
                        echo 'blub'
                    }
                }
            }
        }

        stage('More parallels') {
            parallel {
                stage('Looping') {
                steps {
                    script { 
                        //Loop tests
                        def browsers = ['chrome', 'firefox', 'edge', 'internet explorer']
                        for (int i = 0; i < browsers.size(); ++i) {
                            echo "Testing ${browsers[i]}"
                            stringTest()
                        }
                        }
                    }
                }

                stage('Calling') {
                    steps {
                        script {
                            //Testing if you can pass a def to a method and iterate over it
                            def fruits = ['Apple', 'Banana', 'Grapes', 'Tomato']
                            fruitTest(fruits)
                        }
                }
            }

            }
        }   
            



        }
}

    def fruitTest(fruit) {
        for(int i = 0; i < fruit.size(); ++i) {
            echo "Nom nom, these are the fruits I have: ${fruit[i]}"
        }
        
    }

    def loopTest(browsers) {
        echo "Testing the ${browsers} browser!"
    }

    def stringTest () {
        sh "echo ${blub}"
    }

    def listTest (test) {
        //Returns a string of the list??
        println test
}