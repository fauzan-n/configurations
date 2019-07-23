/* Last Edited: 18/07/2019
 * Version 1.5.1
 * Maintained by: Investree DevOps Team
 */

def slave = "dynamic-slave"
podTemplate(
  label: slave,
  containers:[
    containerTemplate(
      name: "kubectl", image: "gcr.io/cloud-builders/kubectl", command: "cat", ttyEnabled: "true"),
    containerTemplate(
      name: "docker", image: "docker", command: "cat", ttyEnabled: "true"),
    containerTemplate(
      name: "maven", image: "maven:3", command: "cat", ttyEnabled: "true", envVars:[
        secretEnvVar(
          key: 'DB_HOST', secretName: 'database', secretKey: 'host'),
        secretEnvVar(
          key: 'DB_USER', secretName: 'database', secretKey: 'user'),
        secretEnvVar(
          key: 'DB_PASSWD', secretName: 'database', secretKey: 'passwd')])],
  volumes:[
    hostPathVolume(
      hostPath: "/var/run/docker.sock", mountPath: "/var/run/docker.sock"),
    persistentVolumeClaim(
      mountPath: "/root", claimName: "slave-root-home", readOnly: "false")]
){
  node(slave){
    MS_NAME = "test"
    SLACK_CHANNEL = "devops-notifications"
    MS_NAMESPACE = "default"
    DEPLOYMENT_PATH = "/jenkins-home/kubernetes-manifests/deployment/template.yaml"
    DEPLOYMENT = "deployment-template.yaml"
    DOCKERFILE_PATH = "/jenkins-home/dockerfiles/template"
    DOCKERFILE = "dockerfile-template"
    BOOTSTRAP_PATH = "/jenkins-home/spring-cloud-kubernetes-bootstrap/bootstrap.yml"
    BOOTSTRAP= "src/main/resources/bootstrap.yml"
    CONFIGMAP = "configmap.yaml"
    DEV_SET_REPLICA = "2"
    STG_SET_REPLICA = "2"
    PRD_SET_REPLICA = "3"
    DOCKER_REGISTRY = "registry-intl-vpc.ap-southeast-5.aliyuncs.com"
    JENKINS_HOME = "jenkins/jenkins-home"   // Jenkins-home pods
    ALI_REG_CRED_ID = "k8s-deployer"   // Container registry credential ID
    DEV_BRANCH = "origin/develop"   // Develop repository branch name
    STG_BRANCH = "origin/staging"   // Staging repository branch name
    MST_BRANCH = "origin/master"   // Master repository branch name
    CR_DEV_NAMESPACE = "invstr-devel"   // Development container registry namespace
    CR_STG_NAMESPACE = "invstr-staging"   // Staging container registry namespace
    CR_MST_NAMESPACE = "invstr-master"   // Master container registry namespace
    DEV_DOCKER_IMAGE = "${DOCKER_REGISTRY}/${CR_DEV_NAMESPACE}/${MS_NAME}"
    STG_DOCKER_IMAGE = "${DOCKER_REGISTRY}/${CR_STG_NAMESPACE}/${MS_NAME}"
    MST_DOCKER_IMAGE = "${DOCKER_REGISTRY}/${CR_MST_NAMESPACE}/${MS_NAME}"
    ALI_KUBECTL_DEV_CRED_ID = "devel-kubernetes"   // Development kubernetes cluster credential ID
    ALI_KUBECTL_STG_CRED_ID = "staging-kubernetes"   // Staging kubernetes cluster credential ID
    ALI_KUBECTL_PRD_CRED_ID = "production-kubernetes"   // Production kubernetes cluster credential ID
    ALI_KUBE_DEV_SERVER_URL = "https://10.5.1.194:6443"   // Development kubernetes cluster server endpoint
    ALI_KUBE_STG_SERVER_URL = "https://10.7.1.227:6443"   // Staging kubernetes cluster server endpoint
    ALI_KUBE_PRD_SERVER_URL = "https://10.0.0.127:6443"   // Production kubernetes cluster server endpoint
    DEV_DEPLOYMENT = "dev-${MS_NAME}"   // Development deployment name
    STG_DEPLOYMENT = "stg-${MS_NAME}"   // Staging deployment name
    PRD_DEPLOYMENT = "prd-${MS_NAME}"   // Production deployment name

    try{
      slackSend(channel: "#${SLACK_CHANNEL}", color: "good", message: "${MS_NAME} pipeline has been started \n${env.BUILD_URL}")
      stage("Pull Source Code"){
        scmVars = checkout([
          $class: "GitSCM",
          branches: scm.branches,
          userRemoteConfigs: scm.userRemoteConfigs
        ])
        GIT_BRANCH = scmVars.GIT_BRANCH
        echo "REPOSITORY BRANCH : ${GIT_BRANCH}"
        def pom = readMavenPom file: 'pom.xml'
        def version = pom.version.replace("-SNAPSHOT", ".${currentBuild.number}")
        PROJECT_NAME = pom.name
        GROUP_ID = pom.groupId
        MS_VERSION = "${version}"
      }

      stage('Inject Files'){
        container('kubectl'){
          sh "kubectl cp ${JENKINS_HOME}:${BOOTSTRAP_PATH} ${BOOTSTRAP}"
          sh "kubectl cp ${JENKINS_HOME}:${DOCKERFILE_PATH} ${DOCKERFILE}"
          sh "kubectl cp ${JENKINS_HOME}:${DEPLOYMENT_PATH} ${DEPLOYMENT}"
          sh "kubectl apply -f ${CONFIGMAP}"
        }
      }

      stage('Analyse and Build Code'){
        container('maven'){
          sh "sed -i -e 's/MS_NAME/${MS_NAME}/' ${BOOTSTRAP}"
          withSonarQubeEnv('sonar-investree'){
            sh 'mvn -e clean package sonar:sonar'
          }
        }
        timeout(time: 15, unit: 'MINUTES'){
          def qg = waitForQualityGate()
          if (qg.status != "OK"){
            slackSend channel: "#${SLACK_CHANNEL}", color: "danger", message: "Source code cannot pass the quality gate"
            slackSend channel: "#${SLACK_CHANNEL}", color: "danger", message: "https://sonar.investree.id:9999/dashboard?id=${GROUP_ID}:${PROJECT_NAME}"
            currentBuild.result = "ABORTED"
            error ("Source code doesn't pass the quality gate")
          }
        }
      }

      if (GIT_BRANCH == "${DEV_BRANCH}"){
        stage('Build and Push Docker Image'){
          container('docker'){
            withDockerRegistry([
              credentialsId: "${ALI_REG_CRED_ID}",
              url: "https://${DOCKER_REGISTRY}"
            ]){
              sh "docker build --rm -t ${DEV_DOCKER_IMAGE}:${MS_VERSION} -f ${DOCKERFILE} --no-cache ."
              sh "docker push ${DEV_DOCKER_IMAGE}:${MS_VERSION}"
              sh "docker images -a ${DEV_DOCKER_IMAGE}:${MS_VERSION} -q"
              sh "docker rmi -f \$(docker images -a ${DEV_DOCKER_IMAGE}:${MS_VERSION} -q)"
            }
          }
        }

        stage('Deploy to Development'){
          container('kubectl'){
            withKubeConfig([
              credentialsId:"${ALI_KUBECTL_DEV_CRED_ID}",
              serverUrl:"${ALI_KUBE_DEV_SERVER_URL}"
            ]){
              LIST_DEPLOYMENT = sh(script: "kubectl get deployment -n ${MS_NAMESPACE} | grep \"${DEV_DEPLOYMENT}\" | wc -l", returnStdout: true).trim()
              if (LIST_DEPLOYMENT == "1"){
                sh "kubectl apply -f ${CONFIGMAP}"
                sh "kubectl set image -n ${MS_NAMESPACE} deployment/${DEV_DEPLOYMENT} ${DEV_DEPLOYMENT}=${DEV_DOCKER_IMAGE}:${MS_VERSION}"
                slackSend channel: "#${SLACK_CHANNEL}", color: "good", message: "${MS_NAME} on development has been updated to version ${MS_VERSION}"
              }
              else{
                sh "kubectl apply -f ${CONFIGMAP}"
                sh "sed -i -e 's/CR_NAMESPACE/${CR_DEV_NAMESPACE}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/SET_REPLICA/${DEV_SET_REPLICA}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/DEPLOYMENT/${DEV_DEPLOYMENT}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/MS_NAMESPACE/${MS_NAMESPACE}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/MS_VERSION/${MS_VERSION}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/MS_NAME/${MS_NAME}/' ${DEPLOYMENT}"
                sh "kubectl create -f ${DEPLOYMENT}"
                slackSend channel: "#${SLACK_CHANNEL}", color: "good", message: "${MS_NAME}-${MS_VERSION} has been deployed to development"
              }
              slackSend channel: "#${SLACK_CHANNEL}", color: "good", message: "https://dev-svc.investree.id/validate/${MS_NAME}/"
            }
          }
        }
      }

      else if (GIT_BRANCH == "${STG_BRANCH}"){
        stage('Build and Push Docker Image'){
          container('docker'){
            withDockerRegistry([
              credentialsId: "${ALI_REG_CRED_ID}",
              url: "https://${DOCKER_REGISTRY}"
            ]){
              sh "docker build --rm -t ${STG_DOCKER_IMAGE}:${MS_VERSION} -f ${DOCKERFILE} --no-cache ."
              sh "docker push ${STG_DOCKER_IMAGE}:${MS_VERSION}"
              sh "docker images -a ${STG_DOCKER_IMAGE}:${MS_VERSION} -q"
              sh "docker rmi -f \$(docker images -a ${STG_DOCKER_IMAGE}:${MS_VERSION} -q)"
            }
          }
        }

        stage('Deploy to Staging'){
          container('kubectl'){
            withKubeConfig([
              credentialsId:"${ALI_KUBECTL_STG_CRED_ID}",
              serverUrl:"${ALI_KUBE_STG_SERVER_URL}"
            ]){
              LIST_DEPLOYMENT = sh(script: "kubectl get deployment -n ${MS_NAMESPACE} | grep \"${STG_DEPLOYMENT}\" | wc -l", returnStdout: true).trim()
              if (LIST_DEPLOYMENT == "1"){
                sh "kubectl apply -f ${CONFIGMAP}"
                sh "kubectl set image -n ${MS_NAMESPACE} deployment/${STG_DEPLOYMENT} ${STG_DEPLOYMENT}=${STG_DOCKER_IMAGE}:${MS_VERSION}"
                slackSend channel: "#${SLACK_CHANNEL}", color: "good", message: "${MS_NAME} on staging has been updated to version ${MS_VERSION}"
              }
              else{
                sh "kubectl apply -f ${CONFIGMAP}"
                sh "sed -i -e 's/CR_NAMESPACE/${CR_STG_NAMESPACE}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/SET_REPLICA/${STG_SET_REPLICA}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/DEPLOYMENT/${STG_DEPLOYMENT}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/MS_NAMESPACE/${MS_NAMESPACE}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/MS_VERSION/${MS_VERSION}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/MS_NAME/${MS_NAME}/' ${DEPLOYMENT}"
                sh "kubectl create -f ${DEPLOYMENT}"
                slackSend channel: "#${SLACK_CHANNEL}", color: "good", message: "${MS_NAME}-${MS_VERSION} has been deployed to staging"
              }
              slackSend channel: "#${SLACK_CHANNEL}", color: "good", message: "https://stg-svc.investree.id/validate/${MS_NAME}/"
            }
          }
        }
      }

      else if (GIT_BRANCH == "${MST_BRANCH}"){
        stage('Build and Push Docker Image'){
          container('docker'){
            withDockerRegistry([
              credentialsId: "${ALI_REG_CRED_ID}",
              url: "https://${DOCKER_REGISTRY}"
            ]){
              sh "docker build --rm -t ${MST_DOCKER_IMAGE}:${MS_VERSION} -f ${DOCKERFILE} --no-cache ."
              sh "docker push ${MST_DOCKER_IMAGE}:${MS_VERSION}"
              sh "docker images -a ${MST_DOCKER_IMAGE}:${MS_VERSION} -q"
              sh "docker rmi -f \$(docker images -a ${MST_DOCKER_IMAGE}:${MS_VERSION} -q)"
            }
          }
        }

        stage('Deploy to Production'){
          container('kubectl'){
            slackSend channel: "#${SLACK_CHANNEL}", color: "good", message: "Deploy ${MS_NAME}-${MS_VERSION} to Production?"
            def confirmDialog = "Deploy ${MS_NAME}-${MS_VERSION} to Production?"
            releaseApprover = input message: confirmDialog,submitterParameter: 'releaseApprover'
            echo "${releaseApprover} ${MS_NAME}-${MS_VERSION} to Production"
            withKubeConfig([
              credentialsId:"${ALI_KUBECTL_PRD_CRED_ID}",
              serverUrl:"${ALI_KUBE_PRD_SERVER_URL}"
            ]){
              LIST_DEPLOYMENT = sh(script: "kubectl get deployment -n ${MS_NAMESPACE} | grep \"${PRD_DEPLOYMENT}\" | wc -l", returnStdout: true).trim()
              if (LIST_DEPLOYMENT == "1"){
                sh "kubectl apply -f ${CONFIGMAP}"
                sh "kubectl set image -n ${MS_NAMESPACE} deployment/${PRD_DEPLOYMENT} ${PRD_DEPLOYMENT}=${MST_DOCKER_IMAGE}:${MS_VERSION}"
                slackSend channel: "#${SLACK_CHANNEL}", color: "good", message: "${MS_NAME} on production has been updated to version ${MS_VERSION}"
              }
              else{
                sh "kubectl apply -f ${CONFIGMAP}"
                sh "sed -i -e 's/CR_NAMESPACE/${CR_MST_NAMESPACE}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/SET_REPLICA/${PRD_SET_REPLICA}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/DEPLOYMENT/${PRD_DEPLOYMENT}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/MS_NAMESPACE/${MS_NAMESPACE}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/MS_VERSION/${MS_VERSION}/' ${DEPLOYMENT}"
                sh "sed -i -e 's/MS_NAME/${MS_NAME}/' ${DEPLOYMENT}"
                sh "kubectl create -f ${DEPLOYMENT}"
                slackSend channel: "#${SLACK_CHANNEL}", color: "good", message: "${MS_NAME}-${MS_VERSION} has been deployed to production"
              }
              slackSend channel: "#${SLACK_CHANNEL}", color: "good", message: "https://svc.investree.id/validate/${MS_NAME}/"
            }
          }
        }
      }
    }
    catch (e) {
      slackSend channel: "#${SLACK_CHANNEL}", color: "danger", message: "${MS_NAME} pipeline build ${currentBuild.number} failed"
      throw e
    }
  }
}
