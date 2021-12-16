// define a cleanliness function - these make it easier to juggle 
// many containers and clean up the steps blocks
void gradleSh(String command) {
    container('gradle') {
        sh command
    }
}
void nodejsSh(String command){
    container('nodejs'){
        sh command
    }
}

def setSFDXVarsByBranch(String branchName, varName) {
    def sf_username
    def sf_consumer_key
    switch (branchName) {
        case 'int':
            sf_username = "${env.SF_ITDEVSTAGE_USERNAME}"
            sf_consumer_key = "${env.SF_ITDEVSTAGE_CONSUMER_KEY}"
            break
        case 'sit':
            sf_username = "${env.SF_ITStaging_USERNAME}"
            sf_consumer_key = "${env.SF_ITStaging_CONSUMER_KEY}"
            break
    }
    switch (varName) {
        case 'username':
            return sf_username
        case 'consumer_key':
            return sf_consumer_key
    }
    error("You tried running this for an invalid branch")

}

pipeline {
    agent {
        kubernetes {
            yamlFile 'jenkins-agent.yml'
            podRetention never()
        }
    }
    triggers { pollSCM('H H(0-5) * * *') }


    stages {
        stage('Remove Managed files'){
            when {
                anyOf {
                    branch 'master'
                    branch 'int'
                    branch 'sit'
                }
            }
            steps{
                echo "deleting managed package files"
                gradleSh 'grep -iRl "(hidden)" . | xargs rm'
            }
        }
        stage('SonarQube analysis') {
            when {
                anyOf {
                    branch 'master'
                    branch 'int'
                    branch 'sit'
                }
            }
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    gradleSh 'gradle sonarqube -Dsonar.branch.name=$BRANCH_NAME -Dsonar.login=$SONAR_TOKEN'
                }
            }
        }
        stage('PMD Analysis'){
            when {
                anyOf {
                    branch 'master'
                    branch 'int'
                    branch 'sit'
                }
            }
            steps {
                gradleSh 'wget -q https://github.com/pmd/pmd/releases/download/pmd_releases%2F6.40.0/pmd-bin-6.40.0.zip'
                gradleSh 'unzip pmd-bin-6.40.0.zip && mv pmd-bin-6.40.0 pmd'
                gradleSh 'PMD_JAVA_OPTS="-Dpmd.error_recovery" ./pmd/bin/run.sh pmd -d src/classes -f xml -language apex -R rulesets/apex/ruleset.xml -min 3 -no-cache -failOnViolation false > pmd.xml || true'
            }
        }
        stage ('Jest Test') {
            steps {
                gradleSh '''
                         npm install
                         npm run test:unit:machine || true
                         npm run lint:lwc:machine || true
                         '''
            }
        }
        stage('SFDX Setup'){
            when {
                anyOf {
                    branch 'int'
                    branch 'sit'
                }
            }
            environment {
                CONSUMER_KEY=setSFDXVarsByBranch("${env.BRANCH_NAME}",'consumer_key')
                USERNAME=setSFDXVarsByBranch("${env.BRANCH_NAME}",'username')
                ENV_ALIAS="${env.BRANCH_NAME}"
            }
            steps {
                gradleSh 'dnf install xz -y && wget https://developer.salesforce.com/media/salesforce-cli/sfdx-linux-amd64.tar.xz && mkdir sfdx && tar xf sfdx-linux-amd64.tar.xz -C sfdx --strip-components 1 && ./sfdx/install'
                //gradleSh 'npm install -g sfdx-cli'
                withCredentials([file(credentialsId: 'SF_KEY', variable: 'server_key_file')]) {
                    gradleSh 'sfdx auth:jwt:grant --instanceurl https://test.salesforce.com --clientid $CONSUMER_KEY --jwtkeyfile $server_key_file --username $USERNAME --setalias $ENV_ALIAS'
                    gradleSh 'sfdx config:set -g defaultusername=$USERNAME'
                }
                
            }
        }
        stage('Run Apex Tests') {
            when {
                anyOf {
                    branch 'int'
                    branch 'sit'
                }
            }
            steps {
                gradleSh 'sfdx force:apex:test:run --testlevel RunLocalTests --wait 90 -c -r=junit > apex.xml || true'
            }
        }
    }
    post {
        always {
            echo "Collecting results"
            archiveArtifacts 'apex.xml'
            archiveArtifacts 'pmd.xml'
            recordIssues(tools: [pmdParser(pattern: 'pmd.xml')])
            recordIssues(tools: [esLint(pattern: 'eslint.xml')])
            // Attempt to fix the apex.xml file - sed can use any char as separator
            //gradleSh 'sed -i -e "s:\\<A.+/a>.+?>:support page">:g" apex.xml'
            junit allowEmptyResults: true, checksName: 'LWC', testResults: 'junit.xml'
            junit allowEmptyResults: true, checksName: 'Apex', testResults: 'apex.xml'
        }
    }
}
