/**
This pipeline provides a skeleton to build and deploy apps on Openshift Container Platform

@Author Faisal Masood <fmasood@redhat.com>

*/


/*
Jenkins parameterized values:
namespaceDev 			contains the value of openshift namespace, possible values dev/uat etc
namespaceUat			contains the value of openshift namespace, possible values dev/uat etc
envFolder 				contains the environment folder within the project path (e.g. dev, uat)
*/

/***  Global Variables  ***/
def uniqueApplicationId
def appName
def appNameRaw
def appMajorVersion
def appVersion
def gitCommitId
def noProxyHosts

def LOCATION = "https://github.com/masoodfaisal/simple-sb-mysql-rest.git"

node() {


    stage('Checkout'){

            sh '''
                #git config --global http.sslVerify false
                rm -rf simple-sb-mysql-rest || true
                git clone -b master --single-branch https://github.com/masoodfaisal/simple-sb-mysql-rest.git
                cd simple-sb-mysql-rest
               '''

    }


    stage("Populate variables") {

		appVersion = getVersionFromPom("simple-sb-mysql-rest/pom.xml")

        // Remove -SNAPSHOT from app version if any
        int indexSnapshot = appVersion.indexOf("-SNAPSHOT")
        if (indexSnapshot > 0) {
        	appMajorVersion = appVersion.substring(0, indexSnapshot)
        }
        else {
        	appMajorVersion = appVersion
        }

        // Take only the major version (e.g. 1.0.0 -> 1)
        if (appMajorVersion.indexOf('.') > 0) {
        	appMajorVersion = appMajorVersion.substring(0, appMajorVersion.indexOf('.'))
        }
        appMajorVersion = appMajorVersion.replaceAll("[^A-Za-z0-9]", "")


        // Keep only alphanumeric characters in app name
        appNameRaw = getArtifactIdFromPom("simple-sb-mysql-rest/pom.xml")
		//appNameRaw = readFile('appname.txt').trim()
        appName = appNameRaw.replaceAll("[^A-Za-z0-9]", "")

        println "[appName] ${appName}"
       	println "[appMajorVersion] ${appMajorVersion}"

    	// retrieve GIT_COMMIT ID
    	    	sh """
    	        pwd
    	        cd simple-sb-mysql-rest
    	        pwd
            """

    	gitCommitId = sh(returnStdout: true, script: 'cd simple-sb-mysql-rest; git rev-parse HEAD').trim()

    }

    stage('Build Fat jar'){
        sh """
                #export MAVEN_OPTS=''
                cd simple-sb-mysql-rest
                /usr/local/src/apache-maven/bin/mvn -T5  clean package -DskipTests
            """
    }


    stage('Run Unit Test'){

        sh """
                cd simple-sb-mysql-rest
                /usr/local/src/apache-maven/bin/mvn test
            """

    }


    stage('Run Integration Test'){


        sh """
                echo 'Running Integration Test'
                #mvn failsafe:integration-test failsafe:verify
           """

    }






    stage('Vegeta based stress testing'){
        sh """
            #make sure vegeata is available - https://github.com/tsenart/vegeta
            #echo "curl -H content-type:application/json http://localhost:8080/" | ./vegeta attack -duration=60s -rate=200 -keepalive=false | tee results.bin | ./vegeta report
        """
    }

    stage('Start s2i container build'){
        sh """
                cd simple-sb-mysql-rest
                /var/lib/jenkins/oc login https://master-lb.ocp.devops.pd.ntt.hk --insecure-skip-tls-verify=true --token='eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJqZW5raW5zIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImplbmtpbnMtdG9rZW4tYmpmNjgiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiamVua2lucyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImRlYzk4ZjQ2LTFlMjQtMTFlOS04ZmIxLTAwNTA1NmIwMDNiMiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpqZW5raW5zOmplbmtpbnMifQ.S_q2NXwUNAI7b1Pm7RtkBMWwLinnYMdw0OA2r9DrxHVBCj2_YnrVrFGXMiSZF-L9R9ywbCJXidPfx2IQNDnGdFB7K4AxviL7qFg16xijE1DgV9BuCQjLXIeAHukoyEPtMPGRozkRZkegIpKooJ35GalQEHIdtNiP_alTrLzq7TtW09lz52_ZKEIqVh7Hsa0urLFwOIa7PXkUNb1aW7MGWQ1DHLXX82Q54XdYY1HfUxni5dC3CqMaWeohJ4hyHA-zsieHYlJIqnBon1hHBga5TZbMaNdlF8nR69fGm7pxto3m6QgPtfdi4WcPCQHdC8HmWYkYNzMx3-Z9POhHerfnIg'
                /var/lib/jenkins/oc project ${namespaceDev}
                BUILD_STATUS=\$(/var/lib/jenkins/oc  get buildconfig ${appName}${appMajorVersion} -n ${namespaceDev} | awk '{if(NR>1) print \$1}')
                if [[ \${BUILD_STATUS} != ${appName}${appMajorVersion} ]]; then
                    echo "Build does not exist. Creating BuildConfig/IS."
                    /var/lib/jenkins/oc process -f ./openshift/app-build-template.yml -n ${namespaceDev} -p APP_NAME=${appName} -p APP_MAJOR_VERSION=${appMajorVersion} -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} -p APP_VERSION=${appVersion} -p IMAGE_TAG=${appVersion}   | /var/lib/jenkins/oc  create -n ${namespaceDev} -f -
                else
					/var/lib/jenkins/oc  process -f ./openshift/app-build-template.yml -n ${namespaceDev} -p APP_NAME=${appName} -p APP_MAJOR_VERSION=${appMajorVersion} -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} -p APP_VERSION=${appVersion} -p IMAGE_TAG=${appVersion}  | /var/lib/jenkins/oc  replace -n ${namespaceDev} -f -
                fi
           """

        sh """
                cd simple-sb-mysql-rest
                /var/lib/jenkins/oc  project ${namespaceDev}
                /var/lib/jenkins/oc  start-build ${appName}${appMajorVersion} -n ${namespaceDev} --from-file=./target/${appNameRaw}-${appVersion}.jar --follow
           """

    }


    stage('Deploy configmaps to DEV namespace'){
        sh """
                cd simple-sb-mysql-rest
                /var/lib/jenkins/oc  project ${namespaceDev}
                CONFIG_MAP_STATUS=\$(/var/lib/jenkins/oc  get configmap ${appName}${appMajorVersion}  -n ${namespaceDev} | awk '{if(NR>1) print \$1}')
                if [[ \${CONFIG_MAP_STATUS} != ${appName}${appMajorVersion} ]]; then
                    echo "ConfigMap doesnot exists . Creating ConfigMap"
                    /var/lib/jenkins/oc  create configmap ${appName}${appMajorVersion} -n ${namespaceDev} --from-file=configuration/${envFolder}/application.properties
                else
                    /var/lib/jenkins/oc  create configmap ${appName}${appMajorVersion} -n ${namespaceDev} --from-file=configuration/${envFolder}/application.properties --dry-run -o yaml | /var/lib/jenkins/oc  replace configmap  ${appName}${appMajorVersion} -n ${namespaceDev} -f -
                fi

                /var/lib/jenkins/oc  label --overwrite=true configmap ${appName}${appMajorVersion} -n ${namespaceDev} component=${appName} group=app-group project=${appName} provider=s2i version=${appMajorVersion} appname=${appName} appversion=${appVersion} buildnumber=${BUILD_NUMBER} gitcommitid=${gitCommitId}
           """
    }

    stage('Deploy application to DEV namespace'){

        sh """
                cd simple-sb-mysql-rest
                /var/lib/jenkins/oc  project ${namespaceDev}
                DEPLOY_STATUS=\$(/var/lib/jenkins/oc  get deploymentconfig ${appName}${appMajorVersion}  -n ${namespaceDev} | awk '{if(NR>1) print \$1}')
                if [[ \${DEPLOY_STATUS} != ${appName}${appMajorVersion} ]]; then
                    echo "Deployment doesnot exists already. Creating DeploymentConfig/Svc/Route"
                    /var/lib/jenkins/oc  process -f ./openshift/app-deploy-template.yml -n ${namespaceDev} -p EXTERNAL_NAMESPACE=${namespaceDev} -p APP_NAME=${appName} -p APP_MAJOR_VERSION=${appMajorVersion} -p APP_VERSION=${appVersion} -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} -p MEM_LIMIT=900Mi -p MEM_REQUEST=900Mi -p CPU_LIMIT=900m -p CPU_REQUEST=10m -p NAMESPACE=${namespaceDev} -p NUMBER_OF_REPLICAS=1 -p IMAGE_TAG=${appVersion}   | /var/lib/jenkins/oc  create -n ${namespaceDev} -f -
                    sleep 5
                else
                	/var/lib/jenkins/oc  process -f ./openshift/app-deploy-template.yml -n ${namespaceDev} -p EXTERNAL_NAMESPACE=${namespaceDev} -p APP_NAME=${appName} -p APP_MAJOR_VERSION=${appMajorVersion} -p APP_VERSION=${appVersion} -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} -p MEM_LIMIT=900Mi -p MEM_REQUEST=900Mi -p CPU_LIMIT=900m -p CPU_REQUEST=10m -p NAMESPACE=${namespaceDev} -p NUMBER_OF_REPLICAS=1 -p IMAGE_TAG=${appVersion}   | /var/lib/jenkins/oc  apply -n ${namespaceDev} --force=true -f -
                fi

    			#Autoscale
    			HPA_STATUS=\$(/var/lib/jenkins/oc  get hpa ${appName}${appMajorVersion}  -n ${namespaceDev} | awk '{if(NR>1) print \$1}')
                if [[ \${HPA_STATUS} != ${appName}${appMajorVersion} ]]; then
                	echo "Horizontal pod autoscaler does not exist yet. Creating one..."
                	/var/lib/jenkins/oc  autoscale dc/${appName}${appMajorVersion} -n ${namespaceDev} --min 1 --max 1 --cpu-percent=60
                else
                	echo "Horizontal pod autoscaler already exists. Updating labels..."
           		fi
           		/var/lib/jenkins/oc  label --overwrite=true hpa ${appName}${appMajorVersion} -n ${namespaceDev} component=${appName} group=app-group project=${appName} provider=s2i version=${appMajorVersion} appname=${appName} appversion=${appVersion} buildnumber=${BUILD_NUMBER} gitcommitid=${gitCommitId}

                #deploy the docker image
           		/var/lib/jenkins/oc  rollout status dc/${appName}${appMajorVersion} -n ${namespaceDev}
           """


    }

    stage('Expose deployed Service as Route'){
        sh """
                cd simple-sb-mysql-rest
                /var/lib/jenkins/oc  project ${namespaceDev}
                ROUTE_STATUS=\$(/var/lib/jenkins/oc  get route ${appName}${appMajorVersion}  -n ${namespaceDev} | awk '{if(NR>1) print \$1}')
                if [[ \${ROUTE_STATUS} != ${appName}${appMajorVersion} ]]; then
                    echo "Route not found. First deploy"
                    #oc expose svc ${appName}${appMajorVersion} --name=${appName}${appMajorVersion} --path=${apiURL}/v${appMajorVersion} -n ${namespaceDev}
                    /var/lib/jenkins/oc  create route edge ${appName}${appMajorVersion} --service=${appName}${appMajorVersion}  --port=8080 -n ${namespaceDev}
                fi
                /var/lib/jenkins/oc  label --overwrite=true route ${appName}${appMajorVersion} -n ${namespaceDev} component=${appName} group=app-group project=${appName} provider=s2i version=${appMajorVersion} appname=${appName} appversion=${appVersion} buildnumber=${BUILD_NUMBER} gitcommitid=${gitCommitId}
           """

    }



}


// Convenience Functions to read variables from the pom.xml
//https://jenkins.io/doc/pipeline/steps/pipeline-utility-steps/#readmavenpom-read-a-maven-project-file
def getVersionFromPom(pom) {
    def matcher = readFile(pom) =~ '<version>(.+)</version>'
    matcher ? matcher[0][1] : null
}

def getGroupIdFromPom(pom) {
    def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
    matcher ? matcher[0][1] : null
}

def getArtifactIdFromPom(pom) {
    def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
    matcher ? matcher[0][1] : null
}


def getReleaseFromMetaData(pom) {
    def matcher = readFile(pom) =~ '<release>(.+)</release>'
    matcher ? matcher[0][1] : null
}

