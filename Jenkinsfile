env.BUILD_BRANCH = "master"
env.BUILD_NAME = "devops/employee-microservice-node"
env.CONTAINER_NAME = "devops/employee-microservice-nodenpminstall"
env.DEPLOY_ENV = "DEPLOY"
   
node() {
	def scannerHome = tool 'Scanner'
	stage('Code Checkout') {
		checkout([$class: 'GitSCM', branches: [[name: "$BUILD_BRANCH"]], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'CleanBeforeCheckout']], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '0d7b9d79-2fed-416c-b033-99f5a99f21fd', url: 'https://github.com/harsimran498/employee-microservice-node.git']]])
	}

	stage('Run Test Cases'){
		sh "npm install"
		sh "npm test"
	} 
  
	stage('Sonar Scanner'){
		sh "cd ${workspace}"
		sh "${scannerHome}/bin/sonar-scanner"
	}
  
	stage("Quality Check"){
		sh "sleep 30s"
		withSonarQubeEnv('sonar') {
			env.SONAR_CE_TASK_URL = sh(returnStdout: true, script: """cat ${workspace}/.scannerwork/report-task.txt|grep -a 'ceTaskUrl'|awk -F '=' '{print \$2\"=\"\$3}'""")
        timeout(time: 1, unit: 'MINUTES') {
            sh 'curl $SONAR_CE_TASK_URL -o ceTask.json'
            env.analysisID = sh(returnStdout: true, script: """cat ceTask.json |awk -F 'analysisId' '{print \$2}'|awk -F ':' '{print \$2}'|awk -F '\"' '{print \$2}'""")
            env.qualityGateUrl = env.SONAR_HOST_URL + "/api/qualitygates/project_status?analysisId=" + env.analysisID
            sh 'curl $qualityGateUrl -o qualityGate.json'
            env.qualitygate = sh(returnStdout: true, script: """cat qualityGate.json |awk -F 'status' '{print \$2}'|awk -F ':' '{print \$2}'|awk -F '\"' '{print \$2}'""")
            if (qualitygate.trim().equals("ERROR")) {
              error  "Quality Gate failure"
            }
            echo  "Quality Gate successfull"
        }
      }
	}

	stage('Build Base Image'){
		sh "echo $BUILD_NAME"
		sh "docker build -t $BUILD_NAME . "
	} 
  
    stage('Build Application Image'){
      sh "docker build -t $CONTAINER_NAME -f Dockerfile_App . "
     } 
  
	def userInput
	try {
		userInput = input(
			id: 'Proceed1', message: 'Are you Satisfied with results?', parameters: [
			[$class: 'BooleanParameterDefinition', defaultValue: true, description: '', name: 'Please confirm you want to push this build to Nexus Repository Manager for Docker']
			])
	} catch(err) { // input false
		def user = err.getCauses()[0].getUser()
		userInput = false
		echo "Aborted by: [${user}]"
	}

	node {
		if (userInput == true) {
			// do something
			echo "this was successful"
		} else {
			// do something else
			echo "this was not successful"
			currentBuild.result = 'FAILURE'
		} 
	} 

  
	stage ('Login to Nexus repository') {
		withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: '2e191fab-b375-40c8-b3d8-d29f58c63e8a',
		usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
		sh "docker login -u ${USERNAME} -p ${PASSWORD} 34.73.184.207:8083" 
		}
	}

	stage ('Tag Docker Image') {
		sh "docker image tag $CONTAINER_NAME:latest 34.73.184.207:8083/$CONTAINER_NAME:$DEPLOY_ENV.${BUILD_NUMBER}"
	}

	stage ('Upload to Nexus') {
		sh "docker push 34.73.184.207:8083/$CONTAINER_NAME:$DEPLOY_ENV.${BUILD_NUMBER}"
	}

}
