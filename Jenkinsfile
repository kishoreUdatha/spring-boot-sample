def props = ''
def buildNum = ''
artifactVersion = ''
def stashName = ''
def branchName = env.BRANCH_NAME

pipeline{
  agent none //WE DO NOT NEED A JENKINS BUILD AGENT YET
  options{
    skipDefaultCheckout() //DO NOT AUTOMATICALLY CHECKOUT CODE
    timestamps() //ALL PIPELINE ENTRIES IN JENKINS LOG WILL HAVE TIMESTAMP
  }
  stages{
    stage('Setup'){
      steps{
        script{//RUN SCRIPTED PIPELINE LOGIC
          node('master'){
            setupPipeline()
          }

          if (branchName.startsWith('master')){ //CODE WILL ONLY RUN FOR MASTER BRANCHES
            runMasterBranchPipeline()
          }

          if (branchName.startsWith('feature')){ //CODE WILL ONLY RUN FOR FEATURE BRANCHES
            runFeatureBranchPipeline()
          }

          if(branchName.startsWith('release')){
            runReleaseBranchPipeline()
          }

          if(branchName.startsWith('pr')){
            runPullRequestPipeline()
          }

          if(branchName.startsWith('hotfix')){
            runHotfixBranchPipeline()
          }

          if(branchName.startsWith('development')){
            runDevelopBranchPipeline()
          }
        }
      }
    }
  }
}

//*******************************FUNCTION DECLARATIONS*******************************************************

def setupPipeline(){
  cleanWs notFailBuild: true //CLEAN WORKSPACE
  checkout scm
  props = readYaml file: 'Jenkinsfile.yaml'
}

def runReleaseBranchPipeline(){
  runBuildProcess()
  def selectionType = props.dynamicEnvSelection ? 'dynamic' : 'release'
  def deployEnvs = getDeployEnvironments(selectionType)
  deploy(props,deployEnvs,artifactVersion)
}//END RUN RELEASE PIPELINE

def runReleaseProcess(artifactLocation){
  artifactVersion = askDeployVersion()
  echo "deployVersion = ${artifactVersion}"

  node('master'){
    setup()
    stage ('Retrieve artifacts'){ //RUN THE artifactDownload FUNCTION DEFINED BELOW
      artifactDownload(props,artifactVersion,artifactLocation)
    }
    stash includes: getFilesToStash(props), name: stashName //STASH FILES FOR USE ON NEXT JENKINS NODE
  } //LET NODE GO
} //END RUN RELEASE PROCESS

def askDeployVersion(){
  def deployVersion = ''
  stage('Deploy Version'){
    timeout(time: 10, unit: 'MINUTES') {
      deployVersion = input(message: 'Deploy Version', ok: 'OK', parameters: [[$class: 'TextParameterDefinition', description: 'Enter the version to deploy', name: 'Deploy Version']])
    }
  }
  return deployVersion
}//END ASK DEPLOY VERSION

def runFeatureBranchPipeline(){
  runBuildProcess()
  def selectionType = props.dynamicEnvSelection ? 'dynamic' : 'feature'
  def deployEnvs = getDeployEnvironments(selectionType)
  deploy(props,deployEnvs,artifactVersion)
} //END RUN BUILD PIPELINE

def runMasterBranchPipeline(){
  node('master'){
    setup()

    if(props.fortifyOnboared){ //ONLY INITIATE SCAN IF fortifyOnboared TRUE
      stage('Code Security Scan'){ //RUN THE initiateFortifyScan FUNCTION DEFINED BELOW
        initiateFortifyScan(props)
  		}
    }

    stage('Build'){
      runBuild(props,buildNum) //RUN THE runBuild() FUNCTION DEFINED BELOW
    }

    stage('Code Static Analysis'){
      runSonarAnalysis(props,artifactVersion) //RUN THE runSonarAnalysis() FUNCTION DEFINED BELOW
    }
  }
}//END RUN MASTER BRANCH PIPELINE

def runPullRequestPipeline(){
  node('master'){
    setup()

    stage('Build'){
      runBuild(props,buildNum) //RUN THE runBuild() FUNCTION DEFINED BELOW
    }
  }
}//END RUN PULL REQUEST PIPELINE

def runHotfixBranchPipeline(){
  runBuildProcess()
  def selectionType = props.dynamicEnvSelection ? 'dynamic' : 'hotfix'
  def deployEnvs = getDeployEnvironments(selectionType)
  deploy(props,deployEnvs,artifactVersion)
}//END RUN HOT FIX PIPELINE

def runBuildProcess(){
  node('master'){
    setup()

    if(props.fortifyOnboared){ //ONLY INITIATE SCAN IF fortifyOnboared TRUE
      stage('Code Security Scan'){ //RUN THE initiateFortifyScan FUNCTION DEFINED BELOW
        initiateFortifyScan(props)
      }
    }

    stage('Build'){
	    try {
	      runBuild(props,buildNum) //RUN THE runBuild() FUNCTION DEFINED BELOW
	     currentBuild.result = 'SUCCESS'
	    } catch (Exception err) {
	        currentBuild.result = 'FAILURE'
	        echo "During Build result: ${currentBuild.currentResult}"
			def emailSubject = "App name: ${props.artifactId} - Branch: ${branchName} - Jenkins Build Number: ${buildNum} - ${currentBuild.result}"
			def emailBody = """
			
			Jenkins Build Result: 		${currentBuild.result}<br>
		    Jenkins Job URL: 	${env.JOB_URL}<br>
		    Jenkins Job Name: 	${env.JOB_NAME}<br>
		    Jenkins Build Number: ${buildNum}<br>
		    """
			def sendTo = props.mail
			sendEmail(sendTo, emailSubject, emailBody)
			
	    }
    	echo "During Build result: ${currentBuild.result}"
    }
	
	
	def percentClassCoverageReq = '0' //if less than this value job will be marked as UNSTABLE
	def percentComplexityCoverageReg = '0' //if less than this value job will be marked as UNSTABLE
	def percentInstructionCoverageReg = '0' //if less than this value job will be marked as UNSTABLE
	def percentLineCoverageReg = '0' //if less than this value job will be marked as UNSTABLE
	def percentMethodCoverageReg = '0' //if less than this value job will be marked as UNSTABLE
	def percentBranchCoverageReq = '0' //if less than this value job will be marked as UNSTABLE

	stage('Code Coverage Analysis'){
		step([$class: 'JacocoPublisher', changeBuildStatus: true, maximumBranchCoverage: percentBranchCoverageReq,
		maximumClassCoverage: percentClassCoverageReq, maximumComplexityCoverage: percentComplexityCoverageReg,
		maximumInstructionCoverage: percentInstructionCoverageReg, percentLineCoverageReg: percentLineCoverageReg,
		maximumMethodCoverage: percentMethodCoverageReg])
	}
	
	//THIS BLOCK WILL ABORT BUILD IF JACOCO COVERAGES ARE NOT SATISFIED
	if(currentBuild.result.equalsIgnoreCase('UNSTABLE')){
		echo "JaCoCo report build result ${currentBuild.result}"
		currentBuild.result = 'ABORTED'
		error 'JaCoCo coverages not sufficient.  check the report'
	}
	
    stage('Code Static Analysis'){
      runSonarAnalysis(props,artifactVersion) //RUN THE runSonarAnalysis() FUNCTION DEFINED BELOW
    }

    stash includes: "${props.buildArtifactDir}/**, ${getFilesToStash(props)}", name: stashName //STASH FILES FOR USE ON NEXT JENKINS NODE
  } //LET NODE GO

  if(props.gateOnSonar){
    stage('Sonar Quality Check'){ //DO NOT RUN QUALITY CHECK INSIDE A NODE
        checkSonarQuality(artifactVersion)
    }
  }

  node('master'){
    cleanWs notFailBuild: true //CLEAN WORKSPACE
    unstash name: stashName //RESTORE STASHED FILES
    //stage('Artifact Upload'){ //RUN THE artifactUpload() FUNCTION DEFINED BELOW
      artifactUpload(props,artifactVersion)
    //}

    //stage ('Retrieve artifacts'){ //RUN THE artifactDownload FUNCTION DEFINED BELOW
      def artifactRepo = props.buildArtifactRepoName ? props.buildArtifactRepoName : 'libs-snapshot-local'
      artifactDownload(props,artifactVersion,artifactRepo)
    //}

    stash includes: getFilesToStash(props), name: stashName //STASH FILES FOR USE ON NEXT JENKINS NODE
  } //LET NODE GO
}//END BUILD PROCESS

def runDevelopBranchPipeline(){
  runBuildProcess()
  def selectionType = props.dynamicEnvSelection ? 'dynamic' : 'development'
  def deployEnvs = getDeployEnvironments(selectionType)
  deploy(props,deployEnvs,artifactVersion)
} //END RUN DEVELOP BRANCH PIPELINE

def setup(){
  cleanWs notFailBuild: true //CLEAN WORKSPACE
  checkout scm
  buildNum = getBuildNumber(props)
  stashName = 'build_artifact_yaml_stash'
  branchName = env.BRANCH_NAME.toLowerCase()
  artifactVersion = getArtifactVersion(props)
}//END SETUP

def runBuild(props,buildNum){ //USE mavenBuild GLOBAL FUNCTION TO BUILD CODE
  //mavenBuild doc https://bitbucket.service.edp.t-mobile.com/projects/PEJ/repos/build-maven/browse
  library 'pipeline-build' //REQUIRED TO MAKE mavenBuild GLOBAL FUNCTION AVAILABLE
	mavenBuild{
		pomFileLocation = props.pomFileLocation
		appVersion = props.appVersion
		artifactBuildNumber = buildNum
	}
}//END RUN BUILD

def runSonarAnalysis(props,artifactVersion){ //USE sonar GLOBAL FUCNTON TO SCAN AND ANALYZE SOURCE CODE
  //sonar doc https://bitbucket.service.edp.t-mobile.com/projects/PEJ/repos/sonarscan/browse
  library 'pipeline-build'  //REQUIRED TO MAKE sonar GLOBAL FUNCTION AVAILABLE
  sonar{
 	buildType = 'MAVEN'
    runScan = true
    artifactBuildNumber = artifactVersion
    analysisParameters = "-Dsonar.java.binaries=${props.sonarClassesToScanLocation}"
  }
}//END RUN SONAR ANALYSIS

def checkSonarQuality(artifactVersion){ //USE sonar GLOBAL FUCNTON TO SCAN AND ANALYZE SOURCE CODE
  //sonar doc https://bitbucket.service.edp.t-mobile.com/projects/PEJ/repos/sonarscan/browse
  library 'pipeline-build'  //REQUIRED TO MAKE sonar GLOBAL FUNCTION AVAILABLE
  sonar{
    runQualityGate = true
    artifactBuildNumber = artifactVersion
  }
}//END CHECK SONAR QUALITY

def getBuildNumber(props){
  def buildNum = ''
  library 'pipeline-build'
  if (props.useGitShaBuildNumber){
    buildNum = common.getGitTimeStampDotGitSha()
  }else{
    buildNum = common.getJenkinsBuildNumber()
  }
  return buildNum
}//END GET BUILD NUMBER

def getArtifactVersion(props){
  def version = "${props.appVersion}.${getBuildNumber(props)}"
  return version
}//END GET ARTIFACT VERSION

def artifactUpload(props,artifactVersion){ //USE uploadBuildArtifacts GLOBAL FUNCTION TO UPLOAD ARTIFACTS TO ARTIFACTORY
  //uploadBuildArtifacts doc https://bitbucket.service.edp.t-mobile.com/projects/PEJ/repos/build-artifact-upload/browse
  def serverUrl = props.artifactoryServerURL ? props.artifactoryServerURL : 'https://artifactory.service.edp.t-mobile.com/artifactory'
  def artifactRepo = props.buildArtifactRepoName ? props.buildArtifactRepoName : 'libs-snapshot-local'
  library 'pipeline-artifact' //REQUIRED TO MAKE uploadBuildArtifacts GLOBAL FUNCTION AVAILABLE
  uploadBuildArtifacts{
    groupId = props.groupId.replaceAll('\\.','\\/')
    artifactId = props.artifactId
    versionToUpload = artifactVersion
    artifactoryRepository = artifactRepo
    artifactoryServerURL = serverUrl
    artifacts = ["*.${props.buildArtifactType}",'*.yml']
  }
}//END ARTIFACT UPLOAD

def artifactDownload(props,artifactVersion,artifactRepo){ //USE downloadBuildArtifacts GLOBAL FUNCTION TO RETRIEVE ARTIFACTS FROM ARTIFACTORY
  //downloadBuildArtifacts doc https://bitbucket.service.edp.t-mobile.com/projects/PEJ/repos/build-artifact-download/browse
  def serverUrl = props.artifactoryServerURL ? props.artifactoryServerURL : 'https://artifactory.service.edp.t-mobile.com/artifactory'
  library 'pipeline-artifact' //REQUIRED TO MAKE downloadBuildArtifacts GLOBAL FUNCTION AVAILABLE
	downloadBuildArtifacts{  //RUN downloadBuilsArtifacts FUNCTION FROM pipeline-artifact SHARED LIBRARY
		groupId = props.groupId.replaceAll('\\.','\\/')
		artifactId = props.artifactId
		versionToDownload = artifactVersion
		artifactoryRepository = artifactRepo
		artifactoryServerURL = serverUrl
	}
}//END ARTIFACT DOWNLOAD

def pushToCloudFoundry(props,deployProps,artifactVersion,creds,deployScriptName='none'){ //USE THE pcfDeployLegacy GLOBAL FUNCTION TO DEPLOY CODE TO ARTIFACTORY
  deployScriptName = deployScriptName.equalsIgnoreCase('none') ? deployProps.pcfDeployScriptName : deployScriptName
  //pcfDeployLegacy doc https://bitbucket.service.edp.t-mobile.com/projects/PEJ/repos/deploy-pcf-legacy/browse
  library 'pipeline-deploy'  //REQUIRED TO MAKE pcfDeployLegacy FUNCTION AVAILABLE
  pcfDeployLegacy{
    versionToDeploy = artifactVersion
    scriptName = deployScriptName
    groupId = props.groupId.replaceAll('\\.','\\/')
    buildArtifactType = props.buildArtifactType
    artifactId = props.artifactId
    cfAppURL = deployProps.pcfAppName
    lockName = deployProps.pcfEnvLockName
    loginURL = deployProps.pcfApiUrl
    domainURL = deployProps.pcfDomainURL
    space = deployProps.pcfSpace
    org = deployProps.pcfOrg
    manifest = deployProps.pcfManifestName
    credentials = creds
  }
}//END PUSH TO CLOUD FOUNDRY

def pushToCloudFoundryCustomScript(props,deployProps,artifactVersion,creds){
  //getCustomDeployScript(props)
  unstash name: 'shared-pipeline-files' //THIS STASH CONTAINS THE SHARED CUSTOM DEPLOY SCRIPT FILE IS INSIDE pipeline-build FOLDER
  //ARTIFACTS DOWNLOADED WITH EDP downloadBuildArtifacts FUNCTION DOWNLOADED INTO edp-artifacts FOLDER
  def downLoadedArtifactPath = "edp-artifacts/${props.groupId.replaceAll('\\.','\\/')}/${props.artifactId}/${artifactVersion}/${props.artifactId}-${artifactVersion}.${props.buildArtifactType}"
  def params = "${deployProps.pcfApiUrl} ${deployProps.pcfOrg} ${deployProps.pcfSpace} ${deployProps.pcfAppName} ${deployProps.pcfDomainURL} ${deployProps.pcfManifestName} ${downLoadedArtifactPath}"
	library 'pipeline-deploy' //REQUIRED TO MAKE pcfDeploy GLOBAL FUNCTION AVAILABLE
	pcfDeploy{
	  scriptName = "pipeline-shared/${deployProps.pcfCustomScriptName}" //SHELL SCRIPT TO EXECUTE
	  scriptParameters = params //PARAMETERS TO PASS INTO SHELL SCRIPT
	  credentials = creds //CREDENTIAL WITH AUTHORITY TO PUSH TO PCF
	  lockName = deployProps.pcfEnvLockName //NAME OF LOCK TO CREATE.  ONLY ONE PIPELINE CAN HOLD LOCK FOR THE GIVEN NAME
	}
}

def initiateFortifyScan(props){
  library 'pipeline-build' //REQUIRED TO MAKE fortifyScan FUCNTION AVAILABLE
  fortifyScan { //AVAILABLE IN pipeline-build
    akmid = props.akmid
  }
}//END INITIATE FORTIFY SCAN

def deploy(props,deployEnvs,artifactVersion,promoteBuild=false){
  def loopCounter = 0
  for (def environment : deployEnvs) {
    promoteBuild = promoteBuild && loopCounter == 0 ? true : false
    def deployProps = props."${environment}"
    if (deployProps.pcfCustomScriptName){
      deployApplicationCustomScript(props,artifactVersion,promoteBuild,deployProps,environment)
    }else if (deployProps.pcfRouteSwitchScriptName){
      deploySwitchRouteApproval(props,artifactVersion,promoteBuild,deployProps,environment)
    }else{
      deployAutoSwitch(props,artifactVersion,promoteBuild,deployProps,environment)
    }
    loopCounter ++
	
	performAppHealthCheck(props,artifactVersion,deployProps,environment)
	performAppLevelTest(props,artifactVersion,deployProps,environment)
	}
}//END DEPLOY

def deployAutoSwitch(props,artifactVersion,promoteBuild,deployProps,environment){
  requestApproval(deployProps,"${environment}")  //RUN THE requestApproval FUNCTION DEFINED BELOW
  node('master'){
    unstash name: stashName
    if(promoteBuild){
      def artifactRepo = props.releaseArtifactRepoName ? props.releaseArtifactRepoName : 'libs-release-local'
      promoteArtifact(props,artifactVersion,artifactRepo)
    }
    stage("PCF deployment to ${environment}"){
      def creds = deployProps.pcfDeployCreds ? deployProps.pcfDeployCreds : "pcf-${deployProps.pcfOrg}"
      pushToCloudFoundry(props,deployProps,artifactVersion,creds)
    }
    if(deployProps.pcfMapAdditionalRoute){
      mapAdditionalRoute(deployProps,environment)
    }
     
  } //LET NODE GO
}//END DEPLOY AUTO SWITCH


def performAppHealthCheck(props,artifactVersion,deployProps,environment){
  node('master'){
   if (deployProps.skipHealth == true){
				echo "Health Check skipped for ${props.artifactId} Environment: ${environment}"
					def messageText = "Branch: ${branchName} - Artifact Version: ${artifactVersion} - Jenkins Build Number: ${buildNum} - Job URL: ${env.JOB_URL}"
					def messageFrom = "${deployProps.pcfAppName}"+' SkipHealth'
					def messageEmoji = ':no_good:'				
			}else {
			stage("${environment}"+' - Health Check'){
				echo "Health Check for ${props.artifactId} Environment: ${environment}"
				def healthcheck_URL = "http://${deployProps.pcfAppName}.${deployProps.pcfDomainURL}/health"
				//Adding 1 min wait time
				sleep deployProps.sleep
				try{
					def response = httpRequest "${healthcheck_URL}"
					echo "Health Check successfully completed for ${environment}"
					def messageText = "Branch: ${branchName} - Artifact Version: ${artifactVersion} - Health Check URL: ${healthcheck_URL}"
					def messageFrom = "${deployProps.pcfAppName}"+' Health'
					def messageEmoji = ':running:'
				}catch(Exception ex){
			        echo "${environment} Env Health Check Failed"
					def messageText = "Branch: ${branchName} - Artifact Version: ${artifactVersion} - Health Check URL: ${healthcheck_URL}"
					def messageFrom = "${deployProps.pcfAppName}"+' Health'
					def messageEmoji = ':thumbsdown:'
					input('Do you want to proceed?')
					//currentBuild.result = 'ABORTED'
				}
			}
           }
    
  } //LET NODE GO
}//END PERFORM APP HEALTH CHECK


def performAppLevelTest(props,artifactVersion,deployProps,environment){
  node('master'){
        if (deployProps.skipTest == true){
			echo "App level Test skipped for ${props.artifactId} Environment: ${environment}"
				def messageText = "Branch: ${branchName} - Artifact Version: ${artifactVersion} - Jenkins Build Number: ${buildNum} - Job URL: ${env.JOB_URL}" //slack message
				def messageFrom = "${deployProps.pcfAppName}"+' SkipTest'  //who slack message should come from.  defauls to Jenkins
				def messageEmoji = ':no_good:' //emoji used for sent message  valid values https://www.webpagefx.com/tools/emoji-cheat-sheet/.  defaults to :pencil:			
		}else {
			if (environment.equalsIgnoreCase('Production')){
					echo "OrderManagement DevOps team will validate"
			} else { 
			stage("${environment} PostDeploy AppLevel Test") {
				try{
						build job: '../orders-apptest-automation/feature%2FEDP_Jenkins_2018_Integration', propagate: true, wait: true,
						parameters: [
        					string(name: 'env', value: "${environment}"),
        					string(name: 'Test_NG_XML', value: "${props.artifactId}.xml")
    					]
    					 echo "${environment} for app level test "
    					 echo "${props.artifactId} from props artificatId for Health Check Failed"
						// Block and wait for the remote system to callback
						input('App level test Completed successfully and Validated')
						def messageText = "Branch: ${branchName} - Artifact Version: ${artifactVersion} - Jenkins Build Number: ${buildNum} - Job URL: ${env.JOB_URL}" //slack message
						def messageFrom = "${deployProps.pcfAppName}"+' TestComplete'  //who slack message should come from.  defauls to Jenkins
						def messageEmoji = ':thumbsup:' //emoji used for sent message  valid values https://www.webpagefx.com/tools/emoji-cheat-sheet/.  defaults to :pencil:
						
					}catch(Exception ex){
						echo "${environment} App level Test Failed"
						def messageText = "Branch: ${branchName} - Artifact Version: ${artifactVersion} - Jenkins Build Number: ${buildNum} - Job URL: ${env.JOB_URL}" //slack message
						def messageFrom = "${deployProps.pcfAppName}"+' Test Failed'  //who slack message should come from.  defauls to Jenkins
						def messageEmoji = ':thumbsdown:' //emoji used for sent message  valid values https://www.webpagefx.com/tools/emoji-cheat-sheet/.  defaults to :pencil:
						input('Do you want to proceed?')
						//currentBuild.result = 'ABORTED'
					}
			}							
		}
	}
    
  } //LET NODE GO
}//END PERFORM APP LEVEL TEST

def deploySwitchRouteApproval(props,artifactVersion,promoteBuild,deployProps,environment){
  requestApproval(deployProps,"${environment}")  //RUN THE requestApproval FUNCTION DEFINED BELOW
  node('master'){
    unstash name: stashName
    if(promoteBuild){
      def artifactRepo = props.releaseArtifactRepoName ? props.releaseArtifactRepoName : 'libs-release-local'
      promoteArtifact(props,artifactVersion,artifactRepo)
    }
    stage("PCF deployment to ${environment}"){
      def creds = deployProps.pcfDeployCreds ? deployProps.pcfDeployCreds : "pcf-${deployProps.pcfOrg}"
      pushToCloudFoundry(props,deployProps,artifactVersion,creds)
    }
  } //LET NODE GO

  def inputMessage = 'Press Deploy to Switch Route'
  requestRouteApproval(deployProps,"${environment}",inputMessage)  //RUN THE requestApproval FUNCTION DEFINED BELOW

  node('master'){
    unstash name: stashName
    stage("${environment} Switch Route"){
      def creds = deployProps.pcfDeployCreds ? deployProps.pcfDeployCreds : "pcf-${deployProps.pcfOrg}"
      pushToCloudFoundry(props,deployProps,artifactVersion,creds,deployProps.pcfRouteSwitchScriptName) //PASS IN SWITCH ROUTE SCRIPT NAME
    }
    if(deployProps.pcfMapAdditionalRoute){
      mapAdditionalRoute(deployProps,environment)
    }
  } //LET NODE GO
}//END DEPLOY SWITCH ROUTE APPROVAL

def deployApplicationCustomScript(props,artifactVersion,promoteBuild,deployProps,environment){
  requestApproval(deployProps,"${environment}")  //RUN THE requestApproval FUNCTION DEFINED BELOW
  node('master'){
    unstash name: stashName
    if(promoteBuild){
      def artifactRepo = props.releaseArtifactRepoName ? props.releaseArtifactRepoName : 'libs-release-local'
      promoteArtifact(props,artifactVersion,artifactRepo)
    }
    stage("PCF deployment to ${environment}"){
      def creds = deployProps.pcfDeployCreds ? deployProps.pcfDeployCreds : "pcf-${deployProps.pcfOrg}"
      pushToCloudFoundryCustomScript(props,deployProps,artifactVersion,creds)
    }
  } //LET NODE GO
} //END DEPLOY APPLICATION CUSTOM SCRIPT

def requestApproval(deployProps,envName){
  if(deployProps.deployApprovalRequired){ //ONLY REQUEST APPROVAL IF deployApprovalRequired  FOR THE ENVIRONMENT IS TRUE
    stage("Approval for ${envName}"){
      library 'pipeline-utils'
      approval{
         environmentName = envName
         ldapApprovalGroup = deployProps.ldapApprovalGroup //ONLY MEMBERS OF THIS LDAP GROUP OR THIS NETWORK ID WILL BE ALLOWED TO APPROVE
         approvalTimeout = 10
         mail = deployProps.approverEmail //EMAIL WILL SEND TO THIS EMAIL ADDRESS
      }
    }
  }
}//END REQUEST APPROVAL

def requestRouteApproval(deployProps,envName,inputMessage){
  stage("Switch Route Approval for ${envName}"){
    library 'pipeline-utils'
    approval{
       environmentName = envName
       ldapApprovalGroup = deployProps.ldapApprovalGroup //ONLY MEMBERS OF THIS LDAP GROUP OR THIS NETWORK ID WILL BE ALLOWED TO APPROVE
       mail = deployProps.approverEmail //EMAIL WILL SEND TO THIS EMAIL ADDRESS
       message = inputMessage
    }
  }
}//END REQUEST ROUTE APPROVAL

def mapAdditionalRoute(deployProps,environment){
  stage("Map Additional Route to ${environment}"){
    def pcfLoginUrl = deployProps.pcfApiUrl
    def pcfOrg = deployProps.pcfOrg
    def pcfSpace = deployProps.pcfSpace
    def pcfAppName = deployProps.pcfAppName
    def pcfSecondHostName = deployProps.pcfSecondHostName
    def pcfSecondDomainUrl = deployProps.pcfSecondDomainUrl
    def creds = deployProps.pcfDeployCreds ? deployProps.pcfDeployCreds : "pcf-${deployProps.pcfOrg}"
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "${creds}", usernameVariable: 'USER', passwordVariable: 'PASS']]) {
      sh """
      cf login --skip-ssl-validation -a ${pcfLoginUrl} -u ${USER} -p ${PASS} -o ${pcfOrg} -s ${pcfSpace}
      cf map-route ${pcfAppName} ${pcfSecondDomainUrl} --hostname ${pcfSecondHostName}
      """
    }
  }
    //cf login documentation https://cli.cloudfoundry.org/en-US/cf/login.html
    //cf map route documentation https://cli.cloudfoundry.org/en-US/cf/map-route.html
}//END MAP ADDITIONAL ROUTE

def promoteArtifact(props,buildVersion,promoteToRepoName){
  def serverUrl = props.artifactoryServerURL ? props.artifactoryServerURL : 'https://artifactory.service.edp.t-mobile.com/artifactory'
  stage('Promote Artifact'){
	  steps {
            withAWS(region:'us-west-2',credentials:'<AWS-Staging-Jenkins-Credential-ID>') {
              s3Delete(bucket: 'cicd-release-bucket', path:'**/*')
              s3Upload(bucket: 'cicd-release-bucket', workingDir:'build', includePathPattern:'**/*');
            }
  }
  }
}//END PROMOTE ARTIFACT

def getFilesToStash(props){
  def filesToStash = 'edp-artifacts/**'
  for(def file:props.filesToStash){
    filesToStash += ','
    filesToStash +=  file
  }
  return filesToStash
}

def selectEnvironents(props){
  def deployEnvs = ''
  stage('Select Environments'){
    library 'pipeline-utils'
    deployEnvs = environmentSelector{
      envSelectApprover = ''
    }

    if(props.dynamicEnvAllowedChoices){
      def allowedChoices = props.dynamicEnvAllowedChoices
      for(def environment:deployEnvs){
        if (!allowedChoices.contains(environment)){
          error "${environment} is not an allowed dynamic environment choice"
        }
      }
    }
  }
  return deployEnvs
}


def sendEmail(emailTo, emailSubject, emailBody) {
    emailext(subject: emailSubject, body: emailBody, to: emailTo, mimeType: 'text/html')
}

def getDeployEnvironments(selectionType){
  def deployEnvs = ''
  switch(selectionType){
    case 'dynamic':
      deployEnvs = selectEnvironents(props)
      break;
    case 'feature':
      deployEnvs = props.featureBranchDeployEnvs  //ONLY DEPLOY TO ENVIRONMENTS LISTED UNDER PROJECT.YAML featureBranchDeployEnvs
      break;
    case 'hotfix':
      deployEnvs = props.hotfixBranchDeployEnvs //ONLY DEPLOY TO ENVIRONMENTS LISTED UNDER PROJECT.YAML hotfixBranchDeployEnvs
      break;
    case 'release':
      deployEnvs = props.releaseBranchDeployEnvs //ONLY DEPLOY TO ENVIRONMENTS LISTED UNDER PROJECT.YAML releaseBranchDeployEnvs
      break;
    case 'development':
      deployEnvs = props.developBranchDeployEnvs //ONLY DEPLOY TO ENVIRONMENTS LISTED UNDER PROJECT.YAML developBranchDeployEnvs
      break;
  }
  return deployEnvs
}

return this; //REQUIRED AS LAST LINE IN SHARED PIPELINE FILE
