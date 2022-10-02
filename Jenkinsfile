pipeline {
    options {
        disableConcurrentBuilds()
        gitLabConnection gitLabConnection: 'gitlab', jobCredentialId: 'gitlab_apitoken'
        timeout(activity: true, time: 5)
    }
    
    tools {
        jdk 'jdk8'
        maven '3.6.2'
    }
    agent any

    stages {

        stage('Calculate version'){
        when {
                branch "release/*"
            }
        steps {
            script {
            branchNumber = env.BRANCH_NAME.split("/")[1]
            sh "git fetch --tags"
            oldTag = sh(script: "git describe --tags --abbrev=0 || true", returnStdout: true).trim()
            if (!oldTag) {
                        finalNum = "0"
                    } else {
                        finalNum = (oldTag.tokenize(".")[2].toInteger() + 1).toString()
                    }

                    env.VERSION = branchNumber + "." + finalNum
                    echo env.VERSION
                    withCredentials([usernamePassword(credentialsId: 'aleks_jfrog', passwordVariable: 'password', usernameVariable: 'myUser')]){
                    TELEMETRY_VERSION = sh(returnStdout: true, script: "curl -u $myUser:$password http://artifactory:8082/artifactory/exam-libs-release-local/com/lidar/telemetry/maven-metadata.xml | grep '<version>${branchNumber}' | tail -1 | grep -o '[0-9].[0-9].[0-9]'").trim()
                    ANALYTICS_VERSION = sh(returnStdout: true, script: "curl -u $myUser:$password http://artifactory:8082/artifactory/exam-libs-release-local/com/lidar/analytics/maven-metadata.xml | grep '<version>${branchNumber}' | tail -1 | grep -o '[0-9].[0-9].[0-9]'").trim()
            }
            }

                    sh "mvn versions:set -DnewVersion=$env.VERSION"
                    sh  "mvn versions:set-property -Dproperty=telemetry.version -DnewVersion=${TELEMETRY_VERSION}"
                    sh  "mvn versions:set-property -Dproperty=analytics.version -DnewVersion=${ANALYTICS_VERSION}"           
            }
        }
    
 
    stage('Build') {
        when{
            branch "release/*"
        }
      steps {
          configFileProvider([configFile(fileId: 'exam_maven_settings', variable: 'SETTINGS')]){
                sh "mvn verify -s $SETTINGS"
          }

      }
    }

    stage('E2E tests'){
        when{
            branch "release/*"
        }
        sh "mkdir test"            
            withCredentials([usernamePassword(credentialsId: 'aleks_jfrog', passwordVariable: 'password', usernameVariable: 'myUser')]){
                script{
                    //get tests jar
                sh "curl -u $myUser:$password http://artifactory:8082/artifactory/exam-libs-snapshot-local/com/lidar/simulator/99-SNAPSHOT/simulator-99-20220929.101554-1.jar --output test/simulator.jar"
                }
            }
        withCredentials([string(credentialsId: 'testing_api', variable: 'token')]) {
            sh "curl --header 'PRIVATE-TOKEN: $token' http://gitlab/api/v4/projects/8/repository/files/tests-sanity.txt/raw?ref=main --output test/tests.txt"
            }
        sh "unzip target/leader-product-${env.VERSION}-leader-lidar.zip -d test"
            dir('test'){
                sh "java -cp simulator.jar:analytics-${ANALYTICS_VERSION}.jar:telemetry-${TELEMETRY_VERSION}.jar com.lidar.simulation.Simulator"
            }
            sh "rm -r test"



    }

    stage('Publish') {
        when{
            branch "release/*"
        }
        steps {
            configFileProvider([configFile(fileId: 'exam_maven_settings', variable: 'SETTINGS')]) {
            sh "mvn deploy -s $SETTINGS -Dmaven.test.skip"
            }
        }
    }

    stage('Tag'){
        when {
                branch "release/*"
            }
        steps{
            script{
                    sh "git clean -f -x"
                    sh "git tag -a ${env.VERSION} -m 'version ${env.VERSION}'"
                    sh "git push --tag"
            }
        }


    }

    
    }
}


