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
        stage ('preparations') {
            try {
                def deploySettings = getDeploySettings()
                sh './preparations.sh'
            } catch(err) {
                println(err.getMessage());
                throw err
            }
        }
        stage('Build') {
            sh "./Build.sh"
        }
		stage('Results') {
		    sh "./Results.sh"
	   }
        stage ('Tests') {                
            parallel 'static': {
                sh "bin/grumphp run --testsuite=magento2testsuite"
            },
            'unit': {
                sh "magento/bin/magento dev:test:run unit"
            },
            'integration': {
                sh "magento/bin/magento dev:test:run integration"
            }
        }
        if (deploySettings) {
            stage ('Deploy') {
                if (deploySettings.type && deploySettings.version) {
                    // Deploy specific version to a specifc server (IGR or PRD)
                    sh "mg2-builder release:finish -Drelease.type=${deploySettings.type} -Drelease.version=${deploySettings.version}"
                    sh "ssh ${deploySettings.ssh} 'mg2-deployer release -Drelease.version=${deploySettings.version}'"
                    notifyDeployedVersion(deploySettings.version)
                } else {
                    // Deploy to develop branch into IGR server
                    sh "ssh ${deploySettings.ssh} 'mg2-deployer release'"
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