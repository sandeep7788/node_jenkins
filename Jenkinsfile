def JENKINS_WORKER_LABEL = null
def NODE = any
def GIT_REPO = 'https://github.com/sandeep7788/node_jenkins.git'
def GITLAB_REPO = 'stash.s-mxs.net:7999/cs-misolalfred/misolalfred-chatbot-test-application.git'
def GITLAB_REPO_O = 'stash.s-mxs.net:7999/cs-misolalfred/misolalfred-chatbot-test-origin.git'
def GIT_BRANCH = 'master'
def GIT_USER = 'Milan Heimschild'
def GIT_EMAIL = 'milan.heimschild@extern.s-itsolutions.at'
def noActualChanges = false;

pipeline {
    agent {
        label JENKINS_WORKER_LABEL
    }

    triggers {
      pollSCM 'H/5 * * * *'
    }

    stages {
        stage('Checkout') {
            steps {
                node(NODE) {
                    sh 'rm -Rf prepare'
//                    deleteDir();
                    git branch: GIT_BRANCH, credentialsId: 'CHOPSUEY_JENKINS_SSH', url: "ssh://git@${GIT_REPO}"
                }
            }
        }

        stage('Upgrade dependencies') {
            steps {
                node(NODE) {
                    sh 'npm upgrade'
                    sh 'git add package*'
                    sh "git config user.email \"${GIT_EMAIL}\""
                    sh "git config user.name \"${GIT_USER}\""
                    sh script: 'git commit -m "upgraded dependencies"', returnStatus: true
                }
            }
        }

        stage('Last commit check') {
            steps {
                node(NODE) {
                    script {
                        def lastCommit = sh(script: 'git log -1', returnStdout: true)
                        noActualChanges = lastCommit.contains('Upgrade to ')
                        if (noActualChanges) {
                            currentBuild.result = 'ABORTED'
                            error('No actual changes, skipping build.')
                        }
                    }
                }
            }
        }

        stage('Static analysis and test') {
            when {
                expression {
                    return !noActualChanges
                }
            }
            steps {
                node(NODE) {
                    sh 'npm run lint'
                }
            }
        }

        stage('Build') {
            when {
                expression {
                    return !noActualChanges
                }
            }
            steps {
                node(NODE) {
                    sh 'npm run build'
                    sh 'npm ci --production'
                }
            }
        }

        stage('Create version') {
            when {
                expression {
                    return !noActualChanges
                }
            }
            steps {
                node(NODE) {
                    sh "git config user.email \"${GIT_EMAIL}\""
                    sh "git config user.name \"${GIT_USER}\""
                    sh 'npm version patch -m "Upgrade to %s"'
                    withCredentials([sshUserPrivateKey(credentialsId: 'CHOPSUEY_JENKINS_SSH', keyFileVariable: 'KEY_FILE')]) {
                         sh("GIT_SSH_COMMAND='ssh -i ${KEY_FILE}' git push ssh://git@${GIT_REPO} ${GIT_BRANCH} --tags")
                    }
                }
            }
        }

        stage('Prepare') {
            when {
                expression {
                    return !noActualChanges
                }
            }

            steps {
                node(NODE) {
                    sh 'mkdir -p prepare'
                    dir('prepare') {
                        git branch: GIT_BRANCH, credentialsId: 'CHOPSUEY_JENKINS_SSH', url: "ssh://git@${GITLAB_REPO}"
                        sh "cp ../package.json ./app"
                        sh "cp -R ../dist/* ./app"
                        sh "cp -R ../node_modules ./app"
                        sh "git status"
                        script {
                            def version = readJSON(file: 'app/package.json').version
                            sh "git config user.email \"${GIT_EMAIL}\""
                            sh "git config user.name \"${GIT_USER}\""
                            sh "git checkout -B cs-version/v${version}"  // cs-version/v1.0.1
                            sh "git add -A"
                            sh "git commit -m \"Created version ${version}\""

                            withCredentials([sshUserPrivateKey(credentialsId: 'CHOPSUEY_JENKINS_SSH', keyFileVariable: 'KEY_FILE')]) {
                                sh("GIT_SSH_COMMAND='ssh -i ${KEY_FILE}' git push ssh://git@${GITLAB_REPO} cs-version/v${version} --tags")
                            }
                        }
                    }
                }
            }
        }
        stage('Image') {
            when {
                expression {
                    return !noActualChanges
                }
            }
            steps {
                script {
                    withCredentials([sshUserPrivateKey(credentialsId: 'CHOPSUEY_JENKINS_SSH', keyFileVariable: 'KEY_FILE')]) {
                    def versions = sh(script: "GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=no -i ${KEY_FILE}' git ls-remote -h --sort '-v:refname' ssh://git@${GITLAB_REPO}", returnStdout: true).split('\r?\n')

                    env.APP_VERSION = getVersions(versions)[0]
                    csImageBuild 'API_MISOLALFRED_KEY', 'ecp4microf', 'misolalfred', 'chatbot-test', env.APP_VERSION
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                expression {
                    return !noActualChanges
                }
            }
            environment {
                VERBOSE = 'true'
            }
            steps {
                script {
                  withCredentials([sshUserPrivateKey(credentialsId: 'CHOPSUEY_JENKINS_SSH', keyFileVariable: 'KEY_FILE')]) {
                    def versions_o = sh(script: "GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=no -i ${KEY_FILE}' git ls-remote -h --sort '-v:refname' ssh://git@${GITLAB_REPO_O}", returnStdout: true).split('\r?\n')
                    env.CONFIG_VERSION = getVersions(versions_o)[0]
                    csAppDeploy 'API_MISOLALFRED_KEY', 'ecp4microf', 'misolalfred', 'chatbot-test', 'chatbot-test', 'stable', env.APP_VERSION, env.CONFIG_VERSION
                  }               
                }
            }
        }
    }

    post {
        success {
            emailext to: 'alfred@s-itsolutions.at milan.heimschild@extern.s-itsolutions.at Ralph.Zainlinger@s-itsolutions.at Stephan.Schipal@s-itsolutions.at Silke.Kempf-Letuha@s-itsolutions.at Erik.Stabentheiner@s-itsolutions.at',
              recipientProviders: [[$class: 'DevelopersRecipientProvider']],

              subject: "${JOB_NAME} '${env.APP_VERSION}' (conf.: ${env.CONFIG_VERSION}) deployed to the FAT",

              body: "Please see ${BUILD_URL} for more details"

        }
        
        failure {
            emailext to: 'alfred@s-itsolutions.at milan.heimschild@extern.s-itsolutions.at Stephan.Schipal@s-itsolutions.at',
              recipientProviders: [[$class: 'DevelopersRecipientProvider']],
              subject: "Jenkins CD Pipeline '${JOB_NAME}' (${BUILD_NUMBER}) failed",
              body: "Please go to ${BUILD_URL} and verify the build"
        }
    }
}

def getVersions(versions) {
    return versions.findAll({ it.contains('cs-version') }).collect({ it.split('cs-version/v')[1] })
}








