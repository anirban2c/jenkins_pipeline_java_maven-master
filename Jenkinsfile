node {
    def PROJECT_NAME = "Jenkins_File"
	// Get the maven tool.
   // ** NOTE: This 'M3' maven tool must be configured
   // **       in the global configuration.           
   def mvnHome = tool 'M3'

    // Clean workspace before doing anything
    deleteDir()

    propertiesData = [disableConcurrentBuilds()]
    if (isValidDeployBranch()) {
       propertiesData = propertiesData + parameters([
            choice(choices: 'none\nIGR\nPRD', description: 'Target server to deploy', name: 'deployServer'),
        ])
    }
    properties(propertiesData)

    try {
	   
        stage ('Clone') {
				
			checkout scm
			
			git url: 'https://github.com/subrata-mettle/jenkins_pipeline_java_maven-master.git'
        }
		stage('SonarQube analysis') {
			withSonarQubeEnv('Sonar') {
			  // requires SonarQube Scanner for Maven 3.2+
			 // sh "${mvnHome}/bin/mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar"
			  sh "${mvnHome}/bin/mvn clean package sonar:sonar"
			}
		  }
		stage("Quality Gate"){
		  timeout(time: 1, unit: 'HOURS') { // Just in case something goes wrong, pipeline will be killed after a timeout
			def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
			if (qg.status != 'OK') {
			  error "Pipeline aborted due to quality gate failure: ${qg.status}"
			}
			}
		  }
        stage ('preparations') {
            try {
                def deploySettings = getDeploySettings()
				sh "chmod 755 *.sh"
				echo "DEPLOYED VERSION : Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]': Check console output at '${env.BUILD_URL}' [${env.BUILD_NUMBER}]"
				
                sh "./preparations.sh ${deploySettings} ${mvnHome}"
				
            } catch(err) {
                println(err.getMessage());
                throw err
            }
        }
        stage('Build') {
		
            sh "./Build.sh ${mvnHome}"
			
        }
		stage('Results') {
		
		    junit '**/target/surefire-reports/TEST-*.xml'
			archive 'target/*.jar'
			
	   }
        stage ('Tests') {                
            parallel 'static': {
                sh "./static.sh"
            },
            'unit': {
                sh "./unit.sh"
            },
            'integration': {
                sh "./integration.sh"
            }
        }
        if (deploySettings) {
            stage ('Deploy') {
                if (deploySettings.type && deploySettings.version) {
                    // Deploy specific version to a specifc server (IGR or PRD)
                    sh "ssh ${deploySettings.ssh} "
                    notifyDeployedVersion(deploySettings.version)
                } else {
                    // Deploy to develop branch into IGR server
                    //sh "ssh localhost"
		    echo "DEPLOYED VERSION : Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]': Check console output at '${env.BUILD_URL}' [${env.BUILD_NUMBER}]"
		    notifyDeployedVersion("Master Latest Build")
                }
            }
        }
		
    } catch (err) {
        currentBuild.result = 'FAILED'
        notifyFailed()
        throw err
    }
}

def isValidDeployBranch() {
    branchDetails = getBranchDetails()
    if (branchDetails.type == 'hotfix' || branchDetails.type == 'release') {
        return true
    }
    return false
}

def getBranchDetails() {
    def branchDetails = [:]
    branchData = BRANCH_NAME.split('/')
    if (branchData.size() == 2) {
        branchDetails['type'] = branchData[0]
        branchDetails['version'] = branchData[1]
        return branchDetails
    }
    return branchDetails
}

def getDeploySettings() {
    def deploySettings = [:]
    if (BRANCH_NAME == 'master') { 
        deploySettings['ssh'] = "masteruser@domain-igr.com"
    } else if (params.deployServer && params.deployServer != 'none') {
        branchDetails = getBranchDetails()
        deploySettings['type'] = branchDetails.type
        deploySettings['version'] = branchDetails.version
        if (params.deployServer == 'PRD') {
            deploySettings['ssh'] = "prod_user@domain-prd.com"
        } else if (params.deployServer == 'IGR') {
            deploySettings['ssh'] = "igr_user@domain-igr.com"
        }
    }
    return deploySettings
}

def notifyDeployedVersion(String version) {
  emailext (
      subject: "Deployed: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
      body: "DEPLOYED VERSION '${version}': Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]': Check console output at '${env.BUILD_URL}' [${env.BUILD_NUMBER}]",
      to: "some-success-email@some-domain.com"
    )
}

def notifyFailed() {
  emailext (
      subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
      body: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]': Check console output at '${env.BUILD_URL}' [${env.BUILD_NUMBER}]",
      to: "some-fail-email@some-domain.com"
    )
}