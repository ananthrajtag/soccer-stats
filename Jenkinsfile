#!groovy​

// FULL_BUILD -> true/false build parameter to define if we need to run the entire stack for lab purpose only
//final FULL_BUILD = params.FULL_BUILD
final FULL_BUILD = true
// HOST_PROVISION -> server to run ansible based on provision/inventory.ini
//final HOST_PROVISION = params.HOST_PROVISION

final HOST_PROVISION = "ec2-13-234-65-103.ap-south-1.compute.amazonaws.com"

final GIT_URL = 'https://github.com/ananthrajtag/soccer-stats.git'
final NEXUS_URL = '13.234.65.103:8081'

stage('Build') {
    node('LINUX') {
        git GIT_URL
        withEnv(["PATH+MAVEN=${tool 'Maven3'}/bin"]) {
            if(FULL_BUILD) {
                def pom = readMavenPom file: 'pom.xml'
                sh "mvn -B versions:set -DnewVersion=${pom.version}-${BUILD_NUMBER}"
                sh "mvn -B -Dmaven.test.skip=true clean package"
                stash name: "artifact", includes: "target/soccer-stats-*.war"
            }
        }
    }
}

if(FULL_BUILD) {
    stage('Unit Tests') {   
        node('LINUX') {
            withEnv(["PATH+MAVEN=${tool 'Maven3'}/bin"]) {
                sh "mvn -B clean test"
                stash name: "unit_tests", includes: "target/surefire-reports/**"
            }
        }
    }
}

if(FULL_BUILD) {
    stage('Integration Tests') {
        node('LINUX') {
            withEnv(["PATH+MAVEN=${tool 'Maven3'}/bin"]) {
                sh "mvn -B clean verify -Dsurefire.skip=true"
                stash name: 'it_tests', includes: 'target/failsafe-reports/**'
            }
        }
    }
}


if(FULL_BUILD) {
    stage('Static Analysis') {
        node('LINUX') {
            withEnv(["PATH+MAVEN=${tool 'Maven3'}/bin"]) {
                withSonarQubeEnv('SonarQube'){
                    unstash 'it_tests'
                    unstash 'unit_tests'
                    sh 'mvn sonar:sonar -DskipTests'
                }
            }
        }
    }
}

if(FULL_BUILD) {
    stage('Approval') {
        timeout(time:3, unit:'DAYS') {
            input 'Do I have your approval for deployment?'
        }
    }
}


if(FULL_BUILD) {
    stage('Artifact Upload') {
        node('LINUX') {
            unstash 'artifact'

            def pom = readMavenPom file: 'pom.xml'
            def file = "${pom.artifactId}-${pom.version}"
            def jar = "target/${file}.war"

            sh "cp pom.xml ${file}.pom"

            nexusArtifactUploader artifacts: [
                    [artifactId: "${pom.artifactId}", classifier: '', file: "target/${file}.war", type: 'war'],
                    [artifactId: "${pom.artifactId}", classifier: '', file: "${file}.pom", type: 'pom']
                ], 
                credentialsId: 'nexus', 
                nexusInstanceId: 'nexus3',
                nexusRepositoryId: 'ansible-meetup',
                groupId: "${pom.groupId}", 
                nexusUrl: NEXUS_URL, 
                nexusVersion: 'nexus3', 
                protocol: 'http', 
                repository: 'ansible-meetup', 
                version: "${pom.version}"        
        }
    }
}


stage('Deploy') {
    node('LINUX') {
        def pom = readMavenPom file: "pom.xml"
        def repoPath =  "${pom.groupId}".replace(".", "/") + 
                        "/${pom.artifactId}"

        def version = pom.version

        if(!FULL_BUILD) { //takes the last version from repo
            sh "curl -o metadata.xml -s http://${NEXUS_URL}/repository/ansible-meetup/${repoPath}/maven-metadata.xml"
            version = sh script: 'xmllint metadata.xml --xpath "string(//latest)"',
                         returnStdout: true
        }
        def artifactUrl = "http://${NEXUS_URL}/repository/ansible-meetup/${repoPath}/${version}/${pom.artifactId}-${version}.war"

        withEnv(["ARTIFACT_URL=${artifactUrl}", "APP_NAME=${pom.artifactId}"]) {
            echo "The URL is ${env.ARTIFACT_URL} and the app name is ${env.APP_NAME}"

            // install galaxy roles
            sh "ansible-galaxy install -vvv -r provision/requirements.yml -p provision/roles/"        

            ansiblePlaybook colorized: true, 
            credentialsId: 'ssh-jenkins',
            limit: "${HOST_PROVISION}",
            installation: 'ansible',
            inventory: 'provision/inventory.ini', 
            playbook: 'provision/playbook.yml'
           // sudo: true,
          //  sudoUser: 'ec2-user'
        }
    }
}
