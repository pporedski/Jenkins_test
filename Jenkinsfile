import groovy.json.JsonOutput

node('master') {
    stage('DataProvider') {
        echo "Login admin"
             payload = JsonOutput.toJson([username: "admin", password: "admin"])
             sh "curl -c cookiefile \
                     -X POST \
                     -H 'Content-Type: application/json' \
                     -H 'Cache-Control: no-cache' \
                     -d '${payload}' \
                     http://10.189.1.166:8080/njams/api/usermanagement/authentication/ "

             echo "Create Config"
             payload = JsonOutput.toJson([ component: "Njams", name: "instanceName", value: "E2E", publicAccessible : true])
             sh "curl -b cookiefile -X POST -H 'Content-Type: application/json' -H 'Cache-Control: no-cache'  -d '${payload}' \
                 'http://10.189.1.166:8080/njams/api/configuration' "

            echo "Upload JAR: tibjms.jar..."
            sh "curl -b cookiefile \
                -X POST \
                -H 'Content-Type: multipart/form-data' \
                -H 'Cache-Control: no-cache' \
                -F file=@/tmp/tibjms.jar \
                http://10.189.1.166:8080/njams/api/server/deployments/tibjms.jar "

            echo "Upload JAR: tibjmsadmin.jar..."
            sh "curl -b cookiefile \
                -X POST \
                -H 'Content-Type: multipart/form-data' \
                -H 'Cache-Control: no-cache'\
                -F file=@/tmp/tibjmsadmin.jar \
                http://10.189.1.166:8080/njams/api/server/deployments/tibjmsadmin.jar "

            echo "Deploy JARs..."
            sh "curl -b cookiefile -X POST -H 'Cache-Control: no-cache' http://10.189.1.166:8080/njams/api/server/deploy/ "

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
}