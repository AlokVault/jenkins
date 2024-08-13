#!groovy
def jobName = ''
def buildStatus = ''
def commitDetails = ''
def commitArray = ''
def commitUserEmail = ''
def commitUserName = ''
def buildNumber = ''

pipeline {
    agent none
    environment {

        BUILD_TARGET = "main"
        DOCKER_REGISTRY = 'vault123/images'
        VERSION_APP = "1.1"
        SERVICE_NAME = 'test'
        GITREPO_URL = 'https://gitlab.com/test.git'
    }
    options {
        skipStagesAfterUnstable()
        disableConcurrentBuilds()
    }
    stages {
        stage('getting information') {
            agent {
                any {
                    reuseNode true
                }
            }
            steps {
                script{
                    writeFile file: 'console-output.txt', text: currentBuild.rawBuild.getLog()
                    jobName = env.JOB_NAME
                    buildStatus = currentBuild.result ?: 'UNKNOWN'

                    echo "Job Name: ${jobName}"
                    echo "Build Status: ${buildStatus}"

                    // Get the commit details using git command
                    commitDetails = sh(script: "git log -1 --pretty=format:'%an,%ae'", returnStdout: true).trim()
                    commitArray = commitDetails.split(',')

                    commitUserName = commitArray[0]
                    commitUserEmail = commitArray[1]

                    jobName = env.JOB_NAME
                    buildStatus = currentBuild.result ?: 'UNKNOWN'

                    buildNumber = env.BUILD_NUMBER

                    echo "Job Name: ${jobName}"
                    echo "Build Status: ${buildStatus}"

                    echo "Build number: ${buildNumber}"

                    echo "Commit Username: ${commitUserName}"
                    echo "Commit Email: ${commitUserEmail}"
                }
            }
        }
        stage('Preparation ') {
            agent {
                any {
                    reuseNode true
                }
            }
            steps {
                sh 'echo "Preparation and checkout"'
                sh 'rm -rf *'
//                 sh 'git remote prune ${GITREPO_URL}'
                checkout ([$class: 'GitSCM',
                           branches: [[name: '*/main']],
                           extensions: [],
                           userRemoteConfigs: [[credentialsId: 'gitlab-credential',
                                                url: "${GITREPO_URL}"]]])
            }
        }
        stage('Build') {
            agent {
                docker {
                    image 'maven:3.6.1-jdk-13'
                    args '-v /root/.m2:/root/.m2'
                    reuseNode true
                }
            }
            steps {
                script {
                    sh "mvn clean install -DskipTests"
                }
            }
           post{
                failure{
                    googlechatnotification ( url: 'https://chat.googleapis.com/v1/spaces/AAAAMbM5bMM/messages?key=xxxxxxxxxxxxxxxxxxxxxxxxxx',
                            message: "Maven Build for the microservice:'${SERVICE_NAME}'  was unsuccessful on branch '${BUILD_TARGET}'")
                    emailext subject: "Failed: Jobname:'${jobName}', Microservice:'${SERVICE_NAME}', branch:'${BUILD_TARGET}', build status:'${buildStatus}' " ,
                        body: "The Maven build has failed for the microservice:'${SERVICE_NAME}', branch:'${BUILD_TARGET}' and jobname:'${jobName}'.",
                        attachLog: true,
                        attachmentsPattern: 'console-output.txt',
                        to: "${commitUserEmail}"
                }
            }
        }
        stage('Build Docker Image') {
            agent {
                any {
                    reuseNode true
                }
            }
            steps {
                script {
                    now = new Date()
                    DATE_TAG = now.format("YYMMdd")
                    BUILD_TAG_TMS = "${VERSION_APP}.${DATE_TAG}.${BUILD_NUMBER}-${BUILD_TARGET}"
                    writeFile file: 'TMS.txt', text: BUILD_TAG_TMS
                    DOCKER_IMAGE = "${DOCKER_REGISTRY}/skandhtms/${SERVICE_NAME}:${BUILD_TAG_TMS}"
                    print "${DOCKER_IMAGE}"
                    sh "docker build -t ${DOCKER_IMAGE} .  "
                    archiveArtifacts artifacts: 'TMS.txt', fingerprint: true, onlyIfSuccessful: true

                    BUILD_NUMBER_DEV = "${BUILD_NUMBER}"
                    writeFile file: 'dev_build_number.txt' , text: BUILD_NUMBER_DEV
                    archiveArtifacts artifacts: 'dev_build_number.txt', fingerprint: true, onlyIfSuccessful: true
                }
            }
            post{
                failure{
                    googlechatnotification ( url: 'https://chat.googleapis.com/v1/spaces/AAAAMbM5bMM/messages?key=xxxxxxxxxxxxxxxxxxxxxxxx',
                            message: "The Docker image build for the microservice:'${SERVICE_NAME}',branch:'${BUILD_TARGET}' and jobname:'${jobName}' was unsuccessful")
                    emailext subject: " Failed: Docker Image Build for Branch:'${BUILD_TARGET}', microservice:'${SERVICE_NAME}' and the jobname:'${jobName}',build status:'${buildStatus}'",
                        body: "The Docker image build process failed for the microservice '${SERVICE_NAME}, job '${jobName}' with build status:'${buildStatus}'.",
                        attachLog: true,
                        attachmentsPattern: 'console-output.txt',
                        to: "${commitUserEmail}"
                }
            }
        }
        stage('Publish Docker Image') {
            agent {
                any {
                    reuseNode true
                }
            }
            options {
                skipDefaultCheckout true
            }
            steps {
                withCredentials(
                        [usernamePassword(credentialsId: 'DOCKER_HUB_CREDENTIAL',
                                passwordVariable: 'dockerHubPassword',
                                usernameVariable: 'dockerHubUser')]) {
                    script {
                        DOCKER_IMAGE = "${DOCKER_REGISTRY}/skandhtms/${SERVICE_NAME}:${VERSION_APP}.${DATE_TAG}.${BUILD_NUMBER}-${BUILD_TARGET}"
                        DOCKER_REGISTRY = "${DOCKER_REGISTRY}"
                        sh """
                        docker login ${DOCKER_REGISTRY} -u ${dockerHubUser} -p ${dockerHubPassword}
                        docker push ${DOCKER_IMAGE}
                        """
                    }
                }
            }
            post{
                failure{
                    googlechatnotification ( url: 'https://chat.googleapis.com/v1/spaces/AAAAMbM5bMM/messages?key=xxxxxxxxxxxxxxxxxxxxxxxxxx',
                            message: "Publish Docker Image was un-successful for them microservice:'${SERVICE_NAME}' & branch:${BUILD_TARGET}")
                    emailext subject: "Failed: Docker Image Publishing for the microservice:'${SERVICE_NAME}, Branch '${BUILD_TARGET}'",
                        body: "The Docker image publishing process failed for the microservice :'${SERVICE_NAME}',job:'${jobName}', branch:'${BUILD_TARGET}' with build status:'${buildStatus}'. Please check the logs for more details.",
                        attachLog: true,
                        attachmentsPattern: 'console-output.txt',
                        to: "${commitUserEmail}"
                }
            }
        }
        stage('Update Repository With Build Number TAG') {
            agent {
                any {
                    reuseNode true
                }
            }
            options {
                skipDefaultCheckout true
            }
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'gitlab-credential', gitToolName: 'Default')]) {
                  sh """
                  git config --global --replace-all user.email test@gmail.com
                  git config --global --replace-all user.name Alok Kumar
                  git tag -a ${BUILD_TAG_TMS} -m 'Jenkins ${BUILD_TAG_TMS}'
                  git push ${GITREPO_URL} --tags
                  """
                }
            }
            post{
                failure{
                    googlechatnotification ( url: 'https://chat.googleapis.com/v1/spaces/AAAAMbM5bMM/messages?key=xxxxxxxxxxxxxxxxxxxxxxxxxx',
                            message: "Update Repository With Build Number TAG was un-successful for the microservice '${SERVICE_NAME}', branch '${BUILD_TARGET}'")
                    emailext subject: "Failed: Repository Update with Build Number TAG for the microservice:'${SERVICE_NAME}', Branch:'${BUILD_TARGET}'",
                        body: "The process to update the repository with the build number TAG failed for the microservice: '${SERVICE_NAME}',branch:'${BUILD_TARGET}', job:'${jobName}' with build status:'${buildStatus}'. Please review the logs for more details.",
                        attachLog: true,
                        attachmentsPattern: 'console-output.txt',
                        to: "${commitUserEmail}"
                }
            }
        }
        stage ('google space notification') {
            agent {
                any {

                     reuseNode true
                }
            }
            steps {
                 googlechatnotification ( url: 'https://chat.googleapis.com/v1/spaces/AAAAMbM5bMM/messages?key=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
                        message: "Build was successful for branch ${BUILD_TARGET} with tag ${BUILD_TAG_TMS}"  )
            }
        }
    }

}
