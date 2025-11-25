pipeline {
    agent {
        label "${env.EXT_PIPELINE_AGENT}"
    }
    // Define parameters for the pipeline
    parameters {
	    booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run tests stage?')
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch to build')
	    choice(name: 'ENVIRONMENT', choices: ['dev', 'sit', 'uat' ,'prod'], description: 'Select environment')
	    string(name: 'USERNAME', defaultValue: 'admin', description: 'User name for deployment')
        password(name: 'PASSWORD', defaultValue: '', description: 'Password for deployment')
     }

    environment {
	    BUILD_TOOL = 'Maven'
        MAVEN_HOME = tool name: 'Maven', type: 'maven'
        ANYPOINT_USERNAME = credentials('anypoint-username')
        ANYPOINT_PASSWORD = credentials('anypoint-password')
        APP_NAME = "my-mule-app"
 	    BRANCH_NAME = "dev"
        ENV = "dev"
    }
    stages {

     //extract asset parameters from pom.xml
	stage('Parse POM') {
            steps {
                script {
                    // Read the pom.xml content as text
                    def pomContent = readFile 'pom.xml'
                    
                    // Parse XML using XmlSlurper
                    def pom = new XmlSlurper().parseText(pomContent)
                    
                    // Extract common Maven coordinates
                    def groupId = pom.groupId.text() ?: pom.parent.groupId.text()
                    def assetId = pom.artifactId.text()
                    def version = pom.version.text() ?: pom.parent.version.text()
                    
                    // Extract a custom property (example)
                    def customProperty = pom.properties.'my.asset.param'.text()
                    
                    // Print extracted values
                    echo "GroupId: ${groupId}"
                    echo "ArtifactId: ${artifactId}"
                    echo "Version: ${version}"
                    echo "Custom Property (asset param): ${customProperty}"
                    
                    // Optionally set environment variables for next stages
                    env.groupId = groupId
                    env.assetId = assetId
                    env.version = version
                    env.ASSET_PARAM = customProperty
                }
            }
        }
	
	//Determine the Maven profile based on assetId
	stage('determine-profile-from-pom') {
            steps {
                script {
                    if ( ("${env.assetId}" == "xapi") || ("${env.assetId}" == "xpapi" )) {
                       environment {PROFILE = "-Pexperience"}
            		echo "profile is set to ${env.profile}"

                    } else {
			            environment {PROFILE = "-Psystem"}
            		    echo "profile is set to ${env.profile}"
		 	            }//else
		            }
	             }
	} //stage

	stage('Echo Parameters') {
            steps {
                script {
                    // Access parameters using the 'params' object
                    echo "Username: ${params.USERNAME}"
                    echo "Debug Mode: ${params.DEBUG_MODE}"
                    echo "Environment: ${params.ENVIRONMENT}"
                    echo "Password is set: ${params.PASSWORD?.length() > 0}"
					echo "profile  : env.profile "
					echo "groupId: ${params.groupId}"
					echo "assetId: ${params.assetId}"
					echo "version: ${params.version}"
					
                }
            }
    }//stage
		
	//Check if asset exists

	stage('Check Asset Existence') {
            steps {
                script {
                    // Step 1: Get OAuth2 token
		            if ( env.assetId == "xapi") || ( env.assetId == "xpapi" )) {
			            def response = sh(
                        returnStdout: true,
                        script: """curl -s -X POST \\ https://anypoint.mulesoft.com/accounts/api/v2/oauth2/token \\ -H 'Content-Type: application/json' \\ -d 'client_id=${env.ANYPNT_CLIENT_ID}&client_secret=${env.ANYPNT_CLIENT_SECRET}&grant_type=client_credentials'""").trim()
                    
                        def json = readJSON text: response
                        def token = json.access_token                    
                        if (!token) {
	                        error "Failed to obtain access token from Anypoint Platform."
                        }

		            }

                    // Step 2: Check if asset exists in Exchange
                    def assetUrl = "https://anypoint.mulesoft.com/exchange/api/v2/assets/${env.GROUP_ID}/${env.ASSET_ID}/${env.VERSION}"
                    def assetResponse = sh(
                        returnStatus: true,
                        script: """curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer ${token}" "${assetUrl}" """
                    )
                    
                    if (assetResponse == 200) {
                        echo "Asset ${env.GROUP_ID}:${env.ASSET_ID}:${env.VERSION} exists in Anypoint Exchange."
                    } else {
                        echo "Asset ${env.GROUP_ID}:${env.ASSET_ID}:${env.VERSION} does not exist. HTTP Status: ${assetResponse}"
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }//stage
               
      // deployment based on environment

        stage('Environment-Based Deployment') {
            steps {
                script {
                    switch(github.ref_name) {
                        case 'develop':
                            echo "Deploying to Development environment"
			                def publicUrl=""
          			        if ( (${env.assetId} == *"xapi"* ) || ( ${ env.assetId} == *"xpapi"* ) {
            				    publicUrl="-Danypoint.publicUrl=true"
          			        } else{
            				    publicUrl="-Danypoint.publicUrl=false"
				            }
          			
                  			mvn --settings .mvn/settings.xml --batch-mode --update-snapshots deploy -DskipTest \
                  			-Danypoint.lastMile=${{ vars.ANYPOINT_LASTMILE }} \
                  			-DreleaseChannel=LTS \
                  			-Danypoint.properties.key=${{ env.SECURE_KEY }} \
                  			-Danypoint.clientSecret=${{ env.ANYPOINT_CLIENT_SECRET }} -Danypoint.clientId=${{ enc.ANYPOINT_CLIENT_ID }} \
                  			-Danypoint.properties.env=dev \
                  			-Danypoint.properties.envName=-dev \
                  			-Danypoint.skipDeploymentVerification=true \
                  			-Danypoint.replicas=${{ vars.ANYPOINT_REPLICAS }} \
                  			-Danypoint.vCores=${{ vars.ANYPOINT_VCORES }}  \
                  			-Danypoint.spaceName=ucr-privatespace-nonprod \
                  			-Danypoint.environment=DEV \
                  			-Danypoint.updateStrategy=${{ vars.UPDATE_STRATEGY }} \
                  			-Danypoint.connectedAppId=env.MULE_CONNECTED_APP_CLIENT_ID \
                  			-Danypoint.connectedAppSecret=env.MULE_CONNECTED_APP_CLIENT_SECRET \
                  			-Danypoint.vanityDomain="https://dev-ccr.api.mulesoft.hmrc.gov.uk/${{ vars.ANYPOINT_ENDPOINT }},https://dev-ccr.api.mulesoft.hmrc.gov.uk/${{ vars.ANYPOINT_ENDPOINT }}${{ vars.ANYPOINT_ENDPOINT }}" \$publicUrl \ -DmuleDeploy -DskipTests -Pcp_us ${{ env.profile }}

                             break
                    case 'staging':
                        echo "Deploying to Staging environment"
                        break
                    case 'production':
                        echo "Deploying to Production environment"
                        break 
	                } 
                }
             } //script
            } //step
        }//stages   
    } //stages
                                
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
