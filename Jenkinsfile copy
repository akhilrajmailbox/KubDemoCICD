def DEPLOYMENT_ERR = ""
def MY_NODE_LABEL = "SLAVES"
def DOCKER_IMAGE_NAME = "akhilrajmailbox/httpd-demo"

try {
    node (label: "${MY_NODE_LABEL}") {
        stage('Checkout SCM') {
            checkout scm
            RELEASE_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        }
        stage('Creating Docker image') {
            def dockerCredID = "DockerCreds"
            def dockerRegistry = ""
            def imageContext = "Docker"
            def dockerFile = "Dockerfile"
            DOCKER_IMAGE_TAG = "${env.BUILD_NUMBER}.${RELEASE_SHA}"
            dir("${imageContext}") {
                myImage = docker.build("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}", "-f ${dockerFile} .")
            }
            docker.withRegistry("${dockerRegistry}", "${dockerCredID}") {
                myImage = docker.image("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}")
                myImage.push()
            }
        }
        stage('Updating Deployment Snippet'){
            def MODULE_NAME = sh(script: "echo ${JOB_NAME} | awk -F '/${BRANCH_NAME}' '{print \$1}'", returnStdout: true).trim()
            def MODULE_NAME_LOWER = MODULE_NAME.toLowerCase()
            def K8S_NAMESPACE = "${env.BRANCH_NAME}"
            def SVC_IMG_NAME = "${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
            def fileName = "K8s/Deployment.yml"
            def theFileExists = fileExists fileName
            if (theFileExists) {
                sh """
                    sed -i "s|K8S_NAMESPACE_VALUE|${K8S_NAMESPACE}|g" ${fileName}
                    sed -i "s|SVC_IMG_NAME_VALUE|${SVC_IMG_NAME}|g" ${fileName}
                    sed -i "s|RELEASE_VERSION_VALUE|${DOCKER_IMAGE_TAG}|g" ${fileName}
                    sed -i "s|MODULE_NAME_VALUE|${MODULE_NAME_LOWER}|g" ${fileName}
                    cat ${fileName}
                """
            } else {
                error("${fileName} not found..!")
            }
        }
        stage('Cleanup Process') {
            def K8S_NAMESPACE = "${env.BRANCH_NAME}"
            def fileName = "K8s/Deployment.yml"
            def theFileExists = fileExists fileName
            if (theFileExists) {
                sh """
                    docker rmi -f \$(docker images -aq) || echo "Not found"
                    sleep 5
                    kubectl delete -f ${fileName} || echo "Not found"
                    sleep 5
                    kubectl delete ns ${K8S_NAMESPACE} || echo "Not found"
                    sleep 5
                """
            } else {
                error("${fileName} not found..!")
            }
        }
        stage('Setting up Namespace') {
            def K8S_NAMESPACE = "${env.BRANCH_NAME}"
            def fileName = "K8s/Namespace.yml"
            def theFileExists = fileExists fileName
            if (theFileExists) {
                sh """
                    sed -i "s|K8S_NAMESPACE_VALUE|${K8S_NAMESPACE}|g" ${fileName}
                    cat ${fileName}
                    kubectl apply -f ${fileName}
                    sleep 5
                """
            } else {
                error("${fileName} not found..!")
            }       
        }
        stage('Deploy to Kubernetes'){
            def fileName = "K8s/Deployment.yml"
            def theFileExists = fileExists fileName
            if (theFileExists) {
                sh """
                    kubectl apply -f ${fileName}
                    sleep 10
                """
            } else {
                error("${fileName} not found..!")
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
            currentBuild.description = "Demo Jenkins Job"
        }
    }
}