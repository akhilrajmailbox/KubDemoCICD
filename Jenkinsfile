def DEPLOYMENT_ERR = ""
def MY_NODE_LABEL = "SLAVES"
def DOCKER_IMAGE_NAME = "akhilrajmailbox/httpd-demo"
def FIRST_CLONE_BRANCH = "main"

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
        }
        stage('Trigger Jobs') {
            JOB_NAMES = [
                'KubDemoGreen',
                'KubDemoBlue'
            ]
            for (myJobs in JOB_NAMES) {
                MODULE_NAME_LOWER = myJobs.toLowerCase()
                dir(myJobs) {
                    stage('Setting up env') {
                        RELEASE_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                        REPO_BRANCH_NAME = sh(script: "git name-rev --name-only HEAD", returnStdout: true).split("/")[-1].trim()
                        K8S_NAMESPACE = "${DEPLOY_BRANCH}"
                        DOCKER_IMAGE_TAG = "${MODULE_NAME_LOWER}-${env.BUILD_NUMBER}.${RELEASE_SHA}"
                        SVC_IMG_NAME = "${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                        K8S_NS_YAML_FILE = "K8s/Namespace.yml"
                        K8S_DS_YAML_FILE = "K8s/Deployment.yml"
                        DOCKER_CRED = "DockerCreds"
                        DOCKER_REGISTRY = ""
                        IMAGE_CONTEXT = "Docker"
                        DOCKER_FILE = "Dockerfile"
                    }
                    stage('Checkout Code') {
                        git branch: FIRST_CLONE_BRANCH,
                            credentialsId: '',
                            url: "https://github.com/akhilrajmailbox/${myJobs}.git"
                        sh "git checkout ${DEPLOY_BRANCH} || echo 'No Branch Found..!'"
                    }
                    stage('Creating Docker image') {
                        dir("${IMAGE_CONTEXT}") {
                            myImage = docker.build("${SVC_IMG_NAME}", "-f ${DOCKER_FILE} .")
                        }
                        docker.withRegistry("${DOCKER_REGISTRY}", "${DOCKER_CRED}") {
                            myImage = docker.image("${SVC_IMG_NAME}")
                            myImage.push()
                        }
                    }
                    stage('Updating Deployment Snippet'){
                        def theFileExists = fileExists K8S_DS_YAML_FILE
                        if (theFileExists) {
                            sh """
                                sed -i "s|K8S_NAMESPACE_VALUE|${K8S_NAMESPACE}|g" ${K8S_DS_YAML_FILE}
                                sed -i "s|SVC_IMG_NAME_VALUE|${SVC_IMG_NAME}|g" ${K8S_DS_YAML_FILE}
                                sed -i "s|RELEASE_VERSION_VALUE|${DOCKER_IMAGE_TAG}|g" ${K8S_DS_YAML_FILE}
                                sed -i "s|MODULE_NAME_VALUE|${MODULE_NAME_LOWER}|g" ${K8S_DS_YAML_FILE}
                                sed -i "s|REPO_BRANCH_NAME_VALUE|${REPO_BRANCH_NAME}|g" ${K8S_DS_YAML_FILE}
                                cat ${K8S_DS_YAML_FILE}
                            """
                        } else {
                            error("${K8S_DS_YAML_FILE} not found..!")
                        }
                    }
                    stage('Cleanup Process') {
                        def theFileExists = fileExists K8S_DS_YAML_FILE
                        if (theFileExists) {
                            sh """
                                docker rmi -f \$(docker images -aq) || echo "Not found"
                                sleep 5
                                kubectl delete -f ${K8S_DS_YAML_FILE} || echo "Not found"
                                sleep 5
                            """
                        } else {
                            error("${K8S_DS_YAML_FILE} not found..!")
                        }
                    }
                    stage('Setting up Namespace') {
                        def theFileExists = fileExists K8S_NS_YAML_FILE
                        if (theFileExists) {
                            sh """
                                sed -i "s|K8S_NAMESPACE_VALUE|${K8S_NAMESPACE}|g" ${K8S_NS_YAML_FILE}
                                cat ${K8S_NS_YAML_FILE}
                                kubectl apply -f ${K8S_NS_YAML_FILE}
                                sleep 5
                            """
                        } else {
                            error("${K8S_NS_YAML_FILE} not found..!")
                        }       
                    }
                    stage('Deploy to Kubernetes'){
                        def theFileExists = fileExists K8S_DS_YAML_FILE
                        if (theFileExists) {
                            sh """
                                kubectl apply -f ${K8S_DS_YAML_FILE}
                                sleep 10
                            """
                        } else {
                            error("${K8S_DS_YAML_FILE} not found..!")
                        }
                    }
                }
            }
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
            cleanWs()
        }
    }
}