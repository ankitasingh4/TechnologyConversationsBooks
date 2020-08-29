withEnv(['MAIL_TO=anki.2008@gmail.com',
         'MAIL_FROM=anki.2008@gmail.com',
         'S3_UPLOAD_STATUS=SUCCESS',
         'PRESENT_STATUS=DEV_DEPLOY_SUCCESS'
		 ]) {
    echo "${env.JAVA_HOME}"
    echo "Current branch <${env.BRANCH_NAME}>"
    def buildSuccess = false
    def appNameText = "TechnologyConversationsBooks"
    node() {
        stage('Preparation') {
            echo '\u26AB Delete workspace'
            sh "cd ${WORKSPACE}"
            deleteDir()
            sh "${getEnvSetCommand()}"
            sh "env"
            echo "Get code from github for the latest commit in this PR"
            executeCheckout()
        }
        if(env.CHANGE_ID) {
            stage('PR Commit') {
                try {
                    echo "Pull request detected"
                    getEnvSetCommand()
                    buildSuccess = executeBuild()
                    sh "ls -altR ${WORKSPACE}/"
                    if (buildSuccess) {
                      currentBuild.result = 'SUCCESS'
                    }
                    else {
                     currentBuild.result = 'FAILURE'
                    }
                } catch(Exception ex) {
                    echo "PR Commit Failed, below are the exception details...."
                    echo "ex.toString() - ${ex.toString()}"
                    echo "ex.getMessage() - ${ex.getMessage()}"
                    echo "ex.getStackTrace() - ${ex.getStackTrace()}"
                    PRESENT_STATUS = 'COMMIT_FAILURE'
                    currentBuild.result = "FAILURE"
                }
            }
        }
        if(!env.CHANGE_ID) {
            stage('PR Merge') {
                try {
                    echo "PR merge detected"
                    echo '\u26AB Delete workspace'
                    sh "cd ${WORKSPACE}"
                    deleteDir()
                    getEnvSetCommand()
                    executeCheckout()
                    executeBuild()
                    sh '''
                        cd ${WORKSPACE}
                        ls -altR ${WORKSPACE}/
                    '''
                    PRESENT_STATUS = 'MERGE_SUCCESS'
                    currentBuild.result = "SUCCESS"
                    
                } catch(Exception ex) {
                    echo "PR Merge Failed, below are the exception details...."
                    echo "ex.toString() - ${ex.toString()}"
                    echo "ex.getMessage() - ${ex.getMessage()}"
                    PRESENT_STATUS = 'MERGE_FAILURE'
                    currentBuild.result = "FAILURE"
                }
            }
        }
        stage('Code coverage and SonarQube Analysis upload') {
        
            try {
                withSonarQubeEnv('SonarServer') {
                    sh ' cd ${WORKSPACE} '
                
                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: "${WORKSPACE}/TechnologyConversationsBooks/reports/tests/test", reportFiles: 'index.html', reportName: 'TechnologyConversationsBooks Code Coverage Report', reportTitles: 'TechnologyConversationsBooks report'])
                sh '''
                 sonar-scanner -e -Dsonar.sources=. '-Dsonar.branch=${env.BRANCH_NAME}' -Dsonar.projectName=TechnologyConversationsBooks -Dsonar.projectKey=TCB -Dsonar.java.binaries="${WORKSPACE}/src/main" -Dsonar.java.test.binaries="${WORKSPACE}/srctest/" -Dsonar.coverage.exclusions=**/test/**,**/integrationTest/** -Dsonar.exclusions=**/integrationTest/** -Dsonar.java.coveragePlugin=jacoco -Dsonar.jacoco.reportPaths="${WORKSPACE}/jacoco/test.exec" 
                    '''
            } 
            }catch(Exception ex) {
                echo "SonarQube step Failed, below are the exception details...."
                echo "ex.toString() - ${ex.toString()}"
                echo "ex.getMessage() - ${ex.getMessage()}"
                echo "ex.getStackTrace() - ${ex.getStackTrace()}"
            }
        
    }
        if((!env.CHANGE_ID) && (PRESENT_STATUS == "MERGE_SUCCESS")) {
            stage('Upload to AWS S3 - dev Repo') {
                try {
                    currentBuild.result = uploadArtifactToS3("sanity")                    
                    echo "In Upload to Nexus Dev Repo 1" + currentBuild.result
                    PRESENT_STATUS = 'NEXUS_UPLOAD_DEV_REPO_SUCCESS'
                } catch(Exception ex) {
                    echo "Upload to Nexus - dev Repo (actual repo name is commit) Failed...."
                    echo "ex.toString() - ${ex.toString()}"
                    echo "ex.getMessage() - ${ex.getMessage()}"
                    echo "ex.getStackTrace() - ${ex.getStackTrace()}"
                    PRESENT_STATUS = 'NEXUS_UPLOAD_DEV_REPO_FAILURE'
                    currentBuild.result = "FAILURE"
                }
            }
        }
    }
    
    node(){
        stage('Send Notification') {
            sendBuildStatusNotification()
        }
    }
}
def executeCheckout() {
    checkout scm
}

def boolean executeBuild() {
    def result = false
    try {
        sh '''
            cd ${WORKSPACE}
            
            echo "Generating build"
            cat build.gradle
            export GRADLE_HOME=/opt/gradle-4.7
            ./gradlew build test
        '''
        result = true
    } catch(Exception ex) {
        echo "Build Failed, below are the exception details...."
        echo "ex.toString() - ${ex.toString()}"
        echo "ex.getMessage() - ${ex.getMessage()}"
        echo "ex.getStackTrace() - ${ex.getStackTrace()}"
        result = false
    }
    return result
}
def uploadArtifactToS3(REPO_ID) {
    sh "cd ${WORKSPACE}"
	withAWS(region:'dummyS3Region',credentials:'dummyCredentials') {
	s3Upload(bucket:"dummyBucketName", workingDir:'dist', includePathPattern:'**/*');
}
   
    return "${S3_UPLOAD_STATUS}"
}
def sendBuildStatusNotification(subject="") {
    echo 'Sending notification...'
    def isCurrentBuildSuccessful = (currentBuild.result != 'UNSTABLE' && currentBuild.result != 'FAILURE')
    def prevBuildResult = currentBuild?.getPreviousBuild()?.result
    def mailSubject = "${subject} (${env.BRANCH_NAME} $BUILD_DISPLAY_NAME)"
    if (currentBuild.result == 'ABORTED') {
        mailSubject = "${subject} ABORTED, (${env.BRANCH_NAME} $BUILD_DISPLAY_NAME)"
    } else {
        if (!isCurrentBuildSuccessful) {
            mailSubject = "${subject} FAILED, (${env.BRANCH_NAME} $BUILD_DISPLAY_NAME)"
        } else {
            if(prevBuildResult != 'SUCCESS') {
                mailSubject = "${subject} (${env.BRANCH_NAME} $BUILD_DISPLAY_NAME) is back to normal"
            }
        }
    }
    def consoleLink = "${BUILD_URL}consoleFull"
    def changesLink = "${JOB_URL}changes"
    def scriptsWithErrors
    def mailbody = "<p>${BUILD_URL}</p>"+
                "<p><a href='${consoleLink}'>Console Output</a></p>"+
                "<p><a href='${changesLink}'>Changes</a></p>"
    if(scriptsWithErrors?.trim()) {
        mailbody = "$mailbody<p>Scripts with errors:<br><pre>$scriptsWithErrors</pre></p>"
    }
    mail bcc: '', body: "${mailbody}", cc: '', charset: 'UTF-8', from: "${MAIL_FROM}", mimeType: 'text/html', replyTo: '', subject:  "TechnologyConversationsBooks - ${mailSubject}", to: "${MAIL_TO}"
}

def getEnvSetCommand(){
    env.PATH = sh (script:"echo \$PATH:${MVN_HOME}/bin:/bin:/usr/local/bin:/bin:/usr/bin:/usr/sbin", returnStdout: true)
    setPathCommand = "export PATH=\$PATH}"
    
}
