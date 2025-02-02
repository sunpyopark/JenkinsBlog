class Constants {

    static final String MASTER_BRANCH = 'master'

    static final String QA_BUILD = 'Qa'
    static final String RELEASE_BUILD = 'Release'

    static final String INTERNAL_TRACK = 'internal'
    static final String RELEASE_TRACK = 'release'
}

def getBuildType() {
    switch (env.BRANCH_NAME) {
        case Constants.MASTER_BRANCH:
            return Constants.RELEASE_BUILD
        default:
            return Constants.QA_BUILD
    }
}

def getTrackType() {
    switch (env.BRANCH_NAME) {
        case Constants.MASTER_BRANCH:
            return Constants.RELEASE_TRACK
        default:
            return Constants.INTERNAL_TRACK
    }
}

def isDeployCandidate() {
    return ("${env.BRANCH_NAME}" =~ /(develop|master)/)
}

pipeline {

    agent {
        dockerfile {
            label 'linux-agent'
            filename 'Dockerfile'
            dir 'docker/jenkins'
        }
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
    }

    environment {
        appName = 'jenkins-blog'

        KEY_PASSWORD = credentials('keyPassword')
        KEY_ALIAS = credentials('keyAlias')
        STORE_PATH = credentials('storePath')
        STORE_PASSWORD = credentials('storePassword')
    }

    stages {
        stage('Run Tests') {
            steps {
                echo 'Running Tests'
                script {
                    VARIANT = getBuildType()
                    sh "./gradlew test${VARIANT}UnitTest"
                }
            }
        }

        stage('Checkout Keystore') {
            when { expression { return isDeployCandidate() } }
            steps {
                echo 'Checking out keystore for signing'
                checkout([$class                           : 'GitSCM',
                          branches                         : [[name: '*/master']],
                          doGenerateSubmoduleConfigurations: false,
                          extensions                       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'android-keystore']],
                          submoduleCfg                     : [],
                          userRemoteConfigs                : [[credentialsId: 'GitHub', url: 'https://github.com/sunpyopark/JenkinsBlog.git']]
                ])
            }
        }

        stage('Build Bundle') {
            when { expression { return isDeployCandidate() } }
            steps {
                echo 'Building'
                script {
                    VARIANT = getBuildType()
                    sh "./gradlew -PstorePass=${STORE_PASSWORD} -PfilePath=${env.WORKSPACE}/${STORE_PATH} -Palias=${KEY_ALIAS} -PkeyPass=${KEY_PASSWORD} bundle${VARIANT}"
                }
            }
        }

        stage('Deploy App to Store') {
            when { expression { return isDeployCandidate() } }
            steps {
                echo 'Deploying'
                script {
                    VARIANT = getBuildType()
                    TRACK = getTrackType()

                    if (TRACK == RELEASE_TRACK) {
                        timeout(time: 5, unit: 'MINUTES') {
                            input "Proceed with deployment to ${TRACK}?"
                        }
                    }

                    try {
                        CHANGELOG = readFile(file: 'CHANGELOG.txt')
                    } catch (err) {
                        echo "Issue reading CHANGELOG.txt file: ${err.localizedMessage}"
                        CHANGELOG = ''
                    }

                    androidApkUpload googleCredentialsId: 'play-store-credentials',
                            filesPattern: "**/outputs/bundle/${VARIANT.toLowerCase()}/*.aab",
                            trackName: TRACK,
                            recentChangeList: [[language: 'en-US', text: CHANGELOG]]
                }
            }
        }
    }

    post {
        always {
            deleteDir()
            dir("${workspace}@tmp") {
                deleteDir()
            }
        }
    }
}
