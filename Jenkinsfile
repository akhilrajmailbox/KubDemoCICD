def DEPLOYMENT_ERR = ""
def MY_NODE_LABEL = "SLAVES"
def DOCKER_IMAGE_NAME = "akhilrajmailbox/httpd-demo"

properties([
    buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10')),
    disableConcurrentBuilds(),
    rateLimitBuilds(throttle: [count: 2, durationName: 'minute', userBoost: false]),
    [$class: 'RebuildSettings', autoRebuild: false, rebuildDisabled: false],
    parameters([
        string(defaultValue: 'main', name: 'DEPLOY_BRANCH', trim: true)
    ])
])

try {
    node (label: "${MY_NODE_LABEL}") {
        stage('Checkout SCM') {
            checkout scm
            RELEASE_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        }
    }
} catch (err) {
    node (label: "${MY_NODE_LABEL}") {
        stage('Job error stage') {
            currentBuild.result = 'FAILURE'
            DEPLOYMENT_ERR = "${err}"
            throw err
        }
    }
} finally {
    node (label: "${MY_NODE_LABEL}") {
        stage('Job final stage') {
            currentBuild.displayName = "${env.BUILD_NUMBER}"
            currentBuild.description = "Demo Jenkins Job CICD"
        }
    }
}