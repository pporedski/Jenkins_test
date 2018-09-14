#!/usr/bin/env groovy

import groovy.json.JsonOutput

properties([
    buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10')),
    parameters([
        string(defaultValue: 'vsldocker01', description: 'The target environment for ST on Postgres', name: 'ST_ENV'),
		string(defaultValue: 'vsldocker11', description: 'The target environment for ST on Postgres with ES-5', name: 'ST_PG_ES5_ENV'),
        string(defaultValue: 'vsldocker07', description: 'The target environment for ST on Oracle', name: 'ST_ORACLE_ENV'),
        string(defaultValue: 'vsldocker09', description: 'The target environment for ST on MSSQL', name: 'ST_MSSQL_ENV'),
        string(defaultValue: 'vsltest01', description: 'The target environment for Installer Run and E2E Tests', name: 'INST_ENV'),
		string(defaultValue: 'vsltest02', description: 'The target environment for Installer Run and E2E Tests with ES-5', name: 'INST_ES5_ENV')
    ]), 
    pipelineTriggers([
        //pollSCM('')
    ])
])

node ('master') {
   def mvnHome
   env.JAVA_HOME = tool 'jdk-8u92'
   
   stage('Preparation') { // for display purposes
      // Get the Maven tool.
      // ** NOTE: This 'M3' Maven tool must be configured
      // **       in the global configuration.           
      mvnHome = tool 'Maven 3.2.1'
      echo "Check out from SVN"
      echo "Branch Name: ${env.BRANCH_NAME}"
      checkout changelog: false, scm: [$class: 'SubversionSCM', 
                                       additionalCredentials: [], 
                                       excludedCommitMessages: '', 
                                       excludedRegions: '', 
                                       excludedRevprop: '', 
                                       excludedUsers: '', 
                                       filterChangelog: false, 
                                       ignoreDirPropChanges: false, 
                                       includedRegions: '', 
                                       locations: [[credentialsId: '964d8d5e-46b5-4ec0-8edb-9327c2fc0813', 
                                                    depthOption: 'infinity', 
                                                    ignoreExternalsOption: true, 
                                                    local: '.', 
                                                    remote: "svn://svn.integrationmatters.com/IMPRODUCTS/server/${env.BRANCH_NAME}"]], 
                                       workspaceUpdater: [$class: 'UpdateUpdater']]

      stash includes: 'njams4-st/**', name: 'st-source'
      stash includes: 'pom.xml', name: 'parent-pom'
   }

   stage('Build Root Pom and Indexing IF') {
       echo "Build the root pom"
       sh "'${mvnHome}/bin/mvn' clean deploy -N -Pjenkins-cli,svn-check"

       echo "Build Indexing Interfaces"
       sh "'${mvnHome}/bin/mvn' --projects njams4-indexing clean deploy -Pjenkins-cli,svn-check"
   }

   stage('Build') {
   
       parallel backend: {
            lock(resource: 'integrationTestsES6', inversePrecedence: true) {
              
                dir ('njams4') {
					echo "Build Backend without ES support"
					sh "'${mvnHome}/bin/mvn' clean deploy -U -Pjenkins-cli,svn-check -Dmaven.test.skip=true -Ddocker.skip"
					
					echo "Run backend tests with ES-6 (Unit/IT)"
                    try {
                        sh "'${mvnHome}/bin/mvn' verify -U -Pjenkins-cli,svn-check,sonar -Des-6"
                    } finally {
                        junit 'target/surefire-reports/*.xml'  
                        junit 'target/failsafe-reports/*.xml' 
                    }
              
                    echo "Build Javadoc for Backend"
                    sh "'${mvnHome}/bin/mvn' javadoc:javadoc"
                    publishHTML([allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: false,
                        reportDir: 'target/site/apidocs/',
                        reportFiles: 'index.html',
                        reportName: 'Javadoc',
                        reportTitles: ''])
				
                    withSonarQubeEnv('sonar') {
                      sh "'${mvnHome}/bin/mvn' org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar"
                    }
                }
            }
       },
       frontend:{
            echo "Build Frontend"
            sh "'${mvnHome}/bin/mvn' --projects njams4-frontend clean deploy -Pjenkins-cli,svn-check"
       },
       ES5:{
            echo "Build Indexing ES-5 Implementation"
            sh "'${mvnHome}/bin/mvn' --projects njams4-es5 clean deploy -Pjenkins-cli,svn-check"
       },
       ES6:{
            echo "Build Indexing ES-6 Implementation"
            sh "'${mvnHome}/bin/mvn' --projects njams4-es6 clean deploy -Pjenkins-cli,svn-check"
       }
   }
   
   stage('Build WARs') {
		parallel ES5: {
			echo "Build ES-5 WAR"
			sh "'${mvnHome}/bin/mvn' --projects njams-es5 clean deploy -U -Pjenkins-cli,svn-check"
			archiveArtifacts 'njams-es5/target/njams-es5-*.war'
			stash includes: 'njams-es5/target/njams-es5-*.war', name: 'njamsES5WarFile'
		},
       ES6:{
			echo "Build ES-6 WAR"
			sh "'${mvnHome}/bin/mvn' --projects njams-es6 clean deploy -U -Pjenkins-cli,svn-check"
			archiveArtifacts 'njams-es6/target/njams-es6-*.war'
			stash includes: 'njams-es6/target/njams-es6-*.war', name: 'njamsES6WarFile'
	   }
	   
	   milestone label: 'WAR', ordinal: 1
   }
    
   stage('ES-6 System Tests') {
       parallel Postgres: {
         lock(resource: 'vsldocker01', inversePrecedence: true) {
                node ('master') {
                    echo "Deploy for System Tests"

                    unstash 'st-source'
                    unstash 'njamsES6WarFile'
                    unstash 'parent-pom'

                    echo "Restart all containers"
                    sh "ssh docker@${params.ST_ENV} 'cd ~/docker && svn update'"
					sh "ssh docker@${params.ST_ENV} 'cd ~/docker/jenkins && chmod +x *.sh'"
                    sh "ssh docker@${params.ST_ENV} 'chmod +x ~/docker/jenkins/*.sh'"
                    sh "ssh docker@${params.ST_ENV} 'nohup ~/docker/jenkins/start_WF10_ES6_AMQ_PSQL.sh > foo.out 2> foo.err < /dev/null && cat foo.* '"

                    sleep 120

                    //waitUntil {
                    //    sh 'wget --retry-connrefused --tries=120 --waitretry=1 -q http://{params.ST_ENV}:8080 -O /dev/null'
                    //}

                    pom = readMavenPom file: 'pom.xml'
                    warfile = "njams-es6-${pom.version}.war"

                    echo "Warfile name:${warfile}"

                    sh "scp njams-es6/target/${warfile} docker@${params.ST_ENV}:/home/docker "

                    sh "ssh docker@${params.ST_ENV} 'tar -c ${warfile} | docker exec -i wildfly /bin/tar -C /opt/jboss/wildfly/standalone/deployments -x'"
                    sh "ssh docker@${params.ST_ENV} 'touch ~/docker/${warfile}.dodeploy'"
                    sh "ssh docker@${params.ST_ENV} 'docker cp ~/docker/${warfile}.dodeploy wildfly:/opt/jboss/wildfly/standalone/deployments/'"

                    echo "Wait for deployment"
                    sleep 90

                    echo "Check Deployment Success"
                    def RESULT = sh (script: "curl -X GET --header 'Accept: text/plain' 'http://@${params.ST_ENV}:8080/njams/api/public/version'", returnStdout:true).trim()
                    echo "${RESULT}"
                    if(RESULT.contains('404 - Not Found')) {
                        error 'Deployment failed!'
                    }

                    echo "System Tests"
                    dir ('njams4-st') {
                       try {
                           sh "'${mvnHome}/bin/mvn' install -Pjenkins-cli-wildfly"
                       } finally {
                           junit 'target/failsafe-reports/*.xml'    
                       }
                    }
                }
             }
        },
        Oracle:{
            lock(resource: 'vsldocker07', inversePrecedence: true) {
                node ('vsljenkins02') {
                    echo "Deploy for System Tests on ORACLE"
                     
                    unstash 'st-source'
                    unstash 'njamsES6WarFile'
                    unstash 'parent-pom'

                    echo "Restart all containers"
                    sh "ssh docker@${params.ST_ORACLE_ENV} 'cd ~/docker && svn update'"
					sh "ssh docker@${params.ST_ORACLE_ENV} 'cd ~/docker/jenkins && chmod +x *.sh'"
                    sh "ssh docker@${params.ST_ORACLE_ENV} 'chmod +x ~/docker/jenkins/*.sh'"
                    sh "ssh docker@${params.ST_ORACLE_ENV} 'nohup ~/docker/jenkins/start_WF10_ES6_AMQ_ORACLE.sh > foo.out 2> foo.err < /dev/null && cat foo.* '"

                    sleep 60

                    pom = readMavenPom file: 'pom.xml'
                    warfile = "njams-es6-${pom.version}.war"

                    echo "Warfile name:${warfile}"

                    sh "scp njams-es6/target/${warfile} docker@${params.ST_ORACLE_ENV}:/home/docker "

                    sh "ssh docker@${params.ST_ORACLE_ENV} 'tar -c ${warfile} | docker exec -i im_wildfly_oracle /bin/tar -C /opt/jboss/wildfly/standalone/deployments -x'"
                    sh "ssh docker@${params.ST_ORACLE_ENV} 'touch ~/docker/${warfile}.dodeploy'"
                    sh "ssh docker@${params.ST_ORACLE_ENV} 'docker cp ~/docker/${warfile}.dodeploy im_wildfly_oracle:/opt/jboss/wildfly/standalone/deployments/'"

                    echo "Wait for deployment"
                    sleep 80

                    echo "Check Deployment Success"
                    def RESULT = sh (script: "curl -X GET --header 'Accept: text/plain' 'http://@${params.ST_ORACLE_ENV}:8080/njams/api/public/version'", returnStdout:true).trim()
                    echo "${RESULT}"
                    if(RESULT.contains('404 - Not Found')) {
                        error 'Deployment failed!'
                    }

                    echo "System Tests on ORACLE"
                    dir ('njams4-st') {
                       try {
                           sh "'${mvnHome}/bin/mvn' deploy -Pjenkins-cli-wildfly-oracle"
                       } finally {
                           junit 'target/failsafe-reports/*.xml'    
                       }
                    }
                }
            }
        },
        Mssql:{
            lock(resource: 'vsldocker09', inversePrecedence: true) {
                node ('vsljenkins03') {
                    echo "Deploy for System Tests on MSSQL"

                    unstash 'st-source'
                    unstash 'njamsES6WarFile'
                    unstash 'parent-pom'

                    echo "Restart all containers"
                    sh "ssh docker@${params.ST_MSSQL_ENV} 'cd ~/docker && svn update'"
					sh "ssh docker@${params.ST_MSSQL_ENV} 'cd ~/docker/jenkins && chmod +x *.sh'"
                    sh "ssh docker@${params.ST_MSSQL_ENV} 'chmod +x ~/docker/jenkins/*.sh'"
                    sh "ssh docker@${params.ST_MSSQL_ENV} 'nohup ~/docker/jenkins/start_WF10_ES6_AMQ_MSSQL.sh > foo.out 2> foo.err < /dev/null && cat foo.* '"

                    sleep 60

                    pom = readMavenPom file: 'pom.xml'
                    warfile = "njams-es6-${pom.version}.war"

                    echo "Warfile name:${warfile}"

                    sh "scp njams-es6/target/${warfile} docker@${params.ST_MSSQL_ENV}:/home/docker "

                    sh "ssh docker@${params.ST_MSSQL_ENV} 'tar -c ${warfile} | docker exec -i wildflymssqlst /bin/tar -C /opt/jboss/wildfly/standalone/deployments -x'"
                    sh "ssh docker@${params.ST_MSSQL_ENV} 'touch ~/docker/${warfile}.dodeploy'"
                    sh "ssh docker@${params.ST_MSSQL_ENV} 'docker cp ~/docker/${warfile}.dodeploy wildflymssqlst:/opt/jboss/wildfly/standalone/deployments/'"

                    echo "Wait for deployment"
                    sleep 80
                    
                    echo "Check Deployment Success"
                    def RESULT = sh (script: "curl -X GET --header 'Accept: text/plain' 'http://@${params.ST_MSSQL_ENV}:8080/njams/api/public/version'", returnStdout:true).trim()
                    echo "${RESULT}"
                    if(RESULT.contains('404 - Not Found')) {
                        error 'Deployment failed!'
                    }


                    echo "System Tests on MSSQL"
                    dir ('njams4-st') {
                       try {
                           sh "'${mvnHome}/bin/mvn' install -Pjenkins-cli-wildfly-mssql"
                       } finally {
                           junit 'target/failsafe-reports/*.xml'    
                       }
                    }
                }   
            }
        }
   }

    stage('ES-5 Tests') {
		parallel IT: {
			lock(resource: 'integrationTests', inversePrecedence: true) {
				dir ('njams4') {
					echo "Run backend tests with ES-5 (Unit/IT)"
					sh "'${mvnHome}/bin/mvn' clean verify -U -Pjenkins-cli -Des-5"
				}
			}
		},
		ST: {
         lock(resource: 'vsldocker11', inversePrecedence: true) {
                node ('master') {
                    echo "Deploy for System Tests"

                    unstash 'st-source'
                    unstash 'njamsES5WarFile'
                    unstash 'parent-pom'

                    echo "Restart all containers"
                    sh "ssh docker@${params.ST_PG_ES5_ENV} 'cd ~/docker && svn update'"
					sh "ssh docker@${params.ST_PG_ES5_ENV} 'cd ~/docker/jenkins && chmod +x *.sh'"
                    sh "ssh docker@${params.ST_PG_ES5_ENV} 'chmod +x ~/docker/jenkins/*.sh'"
                    sh "ssh docker@${params.ST_PG_ES5_ENV} 'nohup ~/docker/jenkins/start_WF10_ES5_AMQ_PSQL.sh > foo.out 2> foo.err < /dev/null && cat foo.* '"

                    sleep 120

                    //waitUntil {
                    //    sh 'wget --retry-connrefused --tries=120 --waitretry=1 -q http://{params.ST_PG_ES5_ENV}:8080 -O /dev/null'
                    //}

                    pom = readMavenPom file: 'pom.xml'
                    warfile = "njams-es5-${pom.version}.war"

                    echo "Warfile name:${warfile}"

                    sh "scp njams-es5/target/${warfile} docker@${params.ST_PG_ES5_ENV}:/home/docker "

                    sh "ssh docker@${params.ST_PG_ES5_ENV} 'tar -c ${warfile} | docker exec -i wildfly /bin/tar -C /opt/jboss/wildfly/standalone/deployments -x'"
                    sh "ssh docker@${params.ST_PG_ES5_ENV} 'touch ~/docker/${warfile}.dodeploy'"
                    sh "ssh docker@${params.ST_PG_ES5_ENV} 'docker cp ~/docker/${warfile}.dodeploy wildfly:/opt/jboss/wildfly/standalone/deployments/'"

                    echo "Wait for deployment"
                    sleep 90

                    echo "Check Deployment Success"
                    def RESULT = sh (script: "curl -X GET --header 'Accept: text/plain' 'http://@${params.ST_PG_ES5_ENV}:8080/njams/api/public/version'", returnStdout:true).trim()
                    echo "${RESULT}"
                    if(RESULT.contains('404 - Not Found')) {
                        error 'Deployment failed!'
                    }

                    echo "System Tests"
                    dir ('njams4-st') {
                       try {
                           sh "'${mvnHome}/bin/mvn' install -Pjenkins-cli-wildfly"
                       } finally {
                           junit 'target/failsafe-reports/*.xml'    
                       }
                    }
                }
             }
        }
	 milestone label: 'System Tests', ordinal: 2
   }
   
   stage('Build Installer') {
       echo "Build Installer"
       sh "'${mvnHome}/bin/mvn' --projects njams4-install clean package -U"
       archiveArtifacts 'njams4-install/target/media/njams_*.*'
       milestone label: 'Build Installer', ordinal: 3
   }
   

   lock(label: 'E2ETest', inversePrecedence: true) {
        stage('Run Installer'){
			parallel ES6: {
				echo "Run Installer with ES-6"
				 sh "ssh docker@${params.INST_ENV} 'cd ~/docker && svn update'"
				 sh "ssh docker@${params.INST_ENV} 'rm -f ~/docker/jenkins/installer/njams*.sh'"
				 sh "scp -r njams4-install/target/media/*.sh docker@${params.INST_ENV}:~/docker/jenkins/installer/"
				 sh "ssh docker@${params.INST_ENV} 'chmod +x ~/docker/jenkins/installer/*.sh'"
				 sh "ssh docker@${params.INST_ENV} 'cd ~/docker/jenkins/installer'"
				 sh "ssh docker@${params.INST_ENV} 'nohup ~/docker/jenkins/installer/run-installer.sh > foo.out 2> foo.err < /dev/null && cat foo.* '"
				 sh "ssh docker@${params.INST_ENV} 'cat foo.* '"
			},
			ES5: {
				echo "Run Installer without ES; using external ES-5"
				 sh "ssh docker@${params.INST_ES5_ENV} 'cd ~/docker && svn update'"
				 sh "ssh docker@${params.INST_ES5_ENV} 'rm -f ~/docker/jenkins/installer/njams*.sh'"
				 sh "scp -r njams4-install/target/media/*.sh docker@${params.INST_ES5_ENV}:~/docker/jenkins/installer/"
				 sh "ssh docker@${params.INST_ES5_ENV} 'chmod +x ~/docker/jenkins/installer/*.sh'"
				 sh "ssh docker@${params.INST_ES5_ENV} 'cd ~/docker/jenkins/installer'"
				 sh "ssh docker@${params.INST_ES5_ENV} 'nohup ~/docker/jenkins/installer/run-es5-installer.sh > foo.out 2> foo.err < /dev/null && cat foo.* '"
				 sh "ssh docker@${params.INST_ES5_ENV} 'cat foo.* '"
			}
		}
   
         stage('Prepare Env for E2E Tests') {
             echo "Login admin"
             payload = JsonOutput.toJson([username: "admin", password: "admin"])
             sh "curl -c cookiefile \
                     -X POST \
                     -H 'Content-Type: application/json' \
                     -H 'Cache-Control: no-cache' \
                     -d '${payload}' \
                     http://http://10.189.1.166:8080/njams/api/usermanagement/authentication/ "

             echo "Create Config"
             payload = JsonOutput.toJson([ component: "Njams", name: "instanceName", value: "E2E", publicAccessible : true])
             sh "curl -b cookiefile -X POST -H 'Content-Type: application/json' -H 'Cache-Control: no-cache'  -d '${payload}' \
                 'http://http://10.189.1.166:8080/njams/api/configuration' "

            echo "Upload JAR: tibjms.jar..."
            sh "curl -b cookiefile \
                -X POST \
                -H 'Content-Type: multipart/form-data' \
                -H 'Cache-Control: no-cache' \
                -F file=@/tmp/tibjms.jar \
                http://http://10.189.1.166:8080/njams/api/server/deployments/tibjms.jar "

            echo "Upload JAR: tibjmsadmin.jar..."
            sh "curl -b cookiefile \
                -X POST \
                -H 'Content-Type: multipart/form-data' \
                -H 'Cache-Control: no-cache'\
                -F file=@/tmp/tibjmsadmin.jar \
                http://http://10.189.1.166:8080/njams/api/server/deployments/tibjmsadmin.jar "

            echo "Deploy JARs..."
            sh "curl -b cookiefile -X POST -H 'Cache-Control: no-cache' http://http://10.189.1.166:8080/njams/api/server/deploy/ "

            echo "After deploying WildFly will be automatically restarted so we have to wait a bit before go on and login again"
            sleep 60

             echo "Login admin"
             payload = JsonOutput.toJson([username: "admin", password: "admin"])
             sh "curl -c cookiefile \
                     -X POST \
                     -H 'Content-Type: application/json' \
                     -H 'Cache-Control: no-cache' \
                     -d '${payload}' \
                     http://10.189.1.166:8080/njams/api/usermanagement/authentication/ "

             echo "Stop DP"
             payload = JsonOutput.toJson([start: "false"])
             sh "curl -b cookiefile -X PUT -H 'Content-Type: application/json' -H 'Cache-Control: no-cache'  -d '${payload}' \
                 'http://10.189.1.166:8080/njams/api/lifecycle/DataProviderController' "

             echo "Create JMS Connection"
             payload = JsonOutput.toJson([ "connectionFactory":null,
                     "destination":"njams4.e2e",
                     "name":"JmsConnectione2e",
                     "password":"njams4_e2e",
                     "provider":"EMS",
                     "providerUrl":"tcp://vslems01:7222",
                     "ssl":false,
                     "username":"njams4_e2e"])
             sh "curl -b cookiefile -X POST -H 'Content-Type: application/json' -H 'Cache-Control: no-cache' -d '${payload}' \
                 'http://10.189.1.166:8080/njams/api/jmsconnection' "

             echo "Create Dataprovider"
             payload = JsonOutput.toJson([ name: "DataProvider1", 
               threadCount: 8,
               state: true])
             sh "curl -b cookiefile -X POST -H 'Content-Type: application/json' -H 'Cache-Control: no-cache' -d '${payload}' \
                 'http://10.189.1.166:8080/njams/api/jmsdataproviderconfig' "

             echo "Link DP to JmsConnection"
             sh "curl -b cookiefile -X PUT -H 'Content-Type: application/json' -H 'Cache-Control: no-cache' http://10.189.1.166:8080/njams/api/jmsdataproviderconfig/name/DataProvider1/jmsconnection/name/JmsConnectione2e "

             echo "Start DP"
             payload = JsonOutput.toJson([start: "true"])
             sh "curl -b cookiefile -X PUT -H 'Content-Type: application/json' -H 'Cache-Control: no-cache'  -d '${payload}' http://10.189.1.166:8080/njams/api/lifecycle/DataProviderController "

             sleep 15

             echo "Send Projectmessage"
             sh 'java -jar jms-message-sender-1.0-SNAPSHOT-jar-with-dependencies.jar templates/njams4-e2e-test-projectmessage.xml'

             echo "Wait with tests, so that messages could be loaded"
             sleep 300
        }
        stage('E2E Tests') {
            echo "E2E Tests"
            withEnv(["PATH=${tool 'NodeJS 6.9.1'}/bin:${env.PATH}"]) {
                sh 'node --version'
                dir ('njams4-frontend') {
                    sh 'npm install' 
                    sh "npm run clean"
                    sh "./node_modules/.bin/cross-env NODE_BASE_URL=http://${params.INST_ENV}:8080/njams/#/ npm run e2e:prod"

                    junit 'testresults-protractor/*.xml'
                    publishHTML([allowMissing: false, 
                                 alwaysLinkToLastBuild: false, 
                                 keepAll: false, 
                                 reportDir: 'protractor/reports', 
                                 reportFiles: 'htmlReport.html', 
                                 reportName: 'Protractor TestResults', 
                                 reportTitles: ''])
                }
            }
     
            milestone label: 'E2E Tests', ordinal: 4
        }
   }
}
