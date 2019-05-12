
def getTimeStamp(){
    return sh (script: "date +'%Y%m%d%H%M%S%N' | sed 's/[0-9][0-9][0-9][0-9][0-9][0-9]\$//g'", returnStdout: true);
}
def getEnvVar(String paramName){
    return sh (script: "grep '${paramName}' env_vars/project.properties|cut -d'=' -f2", returnStdout: true).trim();
}
def getTargetEnv(String branchName){
    def deploy_env = 'none';
    switch(branchName) {
        case 'master':
            deploy_env='uat'
        break
        case 'develop':
            deploy_env = 'dev'
        default:
            if(branchName.startsWith('release')){
                deploy_env='sit'
            }
            if(branchName.startsWith('feature')){
                deploy_env='none'
            }
    }
    return deploy_env
}

def getImageTag(String currentBranch)
{
    def image_tag = 'latest'
    if(currentBranch==null){
        image_tag = getEnvVar('IMAGE_TAG')
    }
    if(currentBranch=='master'){
        image_tag= getEnvVar('IMAGE_TAG')
    }
    if(currentBranch.startsWith('release')){
        image_tag = currentBranch.substring(8);
    }
    return image_tag
}
pipeline{

environment {
    GIT_COMMIT_SHORT_HASH = sh (script: "git rev-parse --short HEAD", returnStdout: true)
}

agent any

stages{
    stage('Init'){
        steps{
            //checkout scm;
        script{
        env.BASE_DIR = pwd()
        env.CURRENT_BRANCH = env.BRANCH_NAME
        env.IMAGE_TAG = getImageTag(env.CURRENT_BRANCH)
        env.TIMESTAMP = getTimeStamp();
        env.APP_NAME= getEnvVar('APP_NAME')
        env.IMAGE_NAME = getEnvVar('IMAGE_NAME')
        env.PROJECT_NAME=getEnvVar('PROJECT_NAME')
        env.DOCKER_REGISTRY_URL=getEnvVar('DOCKER_REGISTRY_URL')
        env.RELEASE_TAG = getEnvVar('RELEASE_TAG')
        env.DOCKER_PROJECT_NAMESPACE = getEnvVar('DOCKER_PROJECT_NAMESPACE')
        env.DOCKER_IMAGE_TAG= "${DOCKER_REGISTRY_URL}/${DOCKER_PROJECT_NAMESPACE}/${APP_NAME}:${RELEASE_TAG}"

        }

        }
    }

    stage('Cleanup'){
        steps{
            sh '''
            docker rmi -f $(docker images -f 'dangling=true' -q) || true
            docker rmi -f $(docker images | sed 1,2d | awk '{print $3}') || true
            '''
        }

    }

    stage('Test'){
        steps{
            sh '''
            docker run --mount type=bind,src=$(pwd)/app,target=/usr/src -w /usr/src maven:alpine mvn test
            '''
        }

    }

    stage('Build'){
        steps{
            withEnv(["APP_NAME=${APP_NAME}", "PROJECT_NAME=${PROJECT_NAME}"]){
                sh '''
                docker build -t ${DOCKER_REGISTRY_URL}/${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG} --build-arg APP_NAME=${IMAGE_NAME}  -f app/Dockerfile app/.
                '''
            }
        }
    }
    stage('Publish'){
        steps{
            //withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "${JENKINS_DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWD']])
            //{
            sh '''
            docker push ${DOCKER_REGISTRY_URL}/${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG}
            '''
            //}
        }
    }
    stage('Deploy'){
        steps{
        //withCredentials([file(credentialsId: "${JENKINS_GCLOUD_CRED_ID}", variable: 'JENKINSGCLOUDCREDENTIAL')])
        //{
        sh """

            chmod +x $BASE_DIR/k8s/process_files.sh

            cd $BASE_DIR/k8s/
            ./process_files.sh "${IMAGE_NAME}" "${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG}" "./${IMAGE_NAME}/" ${TIMESTAMP}

            cd $BASE_DIR/k8s/${IMAGE_NAME}/.
            kubectl apply --force=true --all=true --record=true -f $BASE_DIR/k8s/$IMAGE_NAME/
            kubectl rollout status --watch=true --v=8 -f $BASE_DIR/k8s/$IMAGE_NAME/$IMAGE_NAME-deployment.yml

            """
        //}
        }
    }
}

post {
    always {
        echo "Build# ${env.BUILD_NUMBER} - Job: ${env.JOB_NUMBER} status is: ${currentBuild.currentResult}"
        // emailext(attachLog: true,
        // mimeType: 'text/html',
        // body: '''
        // <h2>Build# ${env.BUILD_NUMBER} - Job: ${env.JOB_NUMBER} status is: ${currentBuild.currentResult}</h2>
        // <p>Check console output at &QUOT;<a href='${env.BUILD_URL'>${env.JOB_NAME} - [${env.BUILD_NUMBER}]</a>&QUOT;</p>
        // ''',
        // recipientProviders: [[$class: "FirstFailingBuildSusspectRecipientProvider"]],
        // subject: "Build# ${env.BUILD_NUMBER} - Job: ${env.JOB_NUMBER} status is: ${currentBuild.currentResult}",
        // to: "simon.takite@gmail.com")
    }
}
}
