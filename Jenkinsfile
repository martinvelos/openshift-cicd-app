def app_git_repo       = 'https://github.com/elos-tech/openshift-cicd-app.git'
def name_prefix        = 'cicd'
def components_project = "${name_prefix}-components"
def app_project_dev    = "${name_prefix}-tasks-dev"
def app_project_prod   = "${name_prefix}-tasks-prod"

// Convenience Functions to read variables from the pom.xml
// Do not change anything below this line.
// --------------------------------------------------------
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

// Define Maven Command. Make sure it points to the correct
// settings for our Nexus installation (use the service to
// bypass the router). The file nexus_openshift_settings.xml
// needs to be in the Source Code repository.
def mvnCmd = "mvn -s ./nexus_openshift_settings.xml"
//def mvnCmd = "/opt/rh/rh-maven35/root/usr/bin/mvn -s ./nexus_openshift_settings.xml"
//            //^Tested with realpath, but that didn't work either.

pipeline {
  agent {
    // Run this pipeline on the custom{ Maven Slave pod} which has JDK and Maven installed.
    kubernetes {
      label 'maven-pod'
      cloud 'openshift'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    some-label: some-label-value
spec:
  containers:
  - name: maven
    image: docker-registry.default.svc:5000/${name_prefix}-jenkins/jenkins-agent-appdev
    command:
    - cat
    tty: true
"""
    }//Kubernetes
  }//agent
  stages {
    //To bluntly wrap the original code as script leaving the groovy syntax in it, was my first attempt.

    // Checkout Source Code
    stage('Checkout Source') {
      steps{
        git app_git_repo
      }
    }


  stage("build and test in parallel"){
  parallel {
    // start build and test jobs in parallel
    stage('inspecting our environment') {
      steps{
        //def version    = getVersionFromPom("pom.xml")
        //echo "Building version ${version}"
        // Let's prevent "mvn not found":
        //waitUntil()
	//input 'Do we have tha PATH debuged?'
        //|-or other way to set that up?:
        //sh "sleep 600 && which mvn" 
        
        // Testing:
        sh 'whoami'
        sh 'bash -c "echo hello"'
        sh 'cat /etc/*release'
        sh "${mvnCmd} clean package -DskipTests"
	//echo """did not work: sh "export PATH=${M2_HOME}/bin:${PATH} && ${mvnCmd} clean package -DskipTests" """
      }
    }    
    
    // Build .jar with maven:
    stage('Building Java binary') {
      environment {
          version = getVersionFromPom("pom.xml")
      }
      steps{
        echo "Building version ${getVersionFromPom('pom.xml')}"  
        //script{
        // sh "${mvnCmd} clean package -DskipTests"
        //}
      }
     }
  
    // Using Maven run the unit tests
    stage('Unit Tests') {
      steps{
        echo "Running Unit Tests"
        echo "sh \"${mvnCmd} test"
      }
    }
  }//parallel
  }//build&test wrapping par
  
    // Using Maven call SonarQube for Code Analysis
    stage('Code Analysis') {
      environment {
        version = getVersionFromPom("pom.xml")
	devTag  = "${version}-${BUILD_NUMBER}"
      }
      steps{
        echo "Running Code Analysis"
        echo "sh \"${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube.${components_project}.svc:9000 -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"
      }
    }
  
    // Publish the built war file to Nexus
    stage('Publish to Nexus') {
      steps{
        echo "Publish to Nexus"
        echo "sh \"${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.${components_project}.svc:8081/repository/releases"
      }
    }
  
    // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      environment {
        version = getVersionFromPom("pom.xml")
        // Set the tag for the development image: version + build number
        devTag  = "${version}-${BUILD_NUMBER}"
      }
      steps{
        echo "Building OpenShift container image tasks:${devTag}"
   
      // Use the file we've published into Nexus in the previous stage:
      echo "sh \"oc start-build tasks --follow --from-file=http://nexus3.${components_project}.svc:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${version}/tasks-${version}.war -n ${app_project_dev}"
  
      // Tag the image using the devTag
      //openshiftTag alias: 'false', destStream: 'tasks', destTag: devTag, destinationNamespace: "$app_project_dev", namespace: "$app_project_dev", srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
      }
    }
  
    // Copy Image to Nexus Docker Registry
    stage('Copy Image to Nexus Docker Registry') {
      environment {
        version = getVersionFromPom("pom.xml")
	devTag  = "${version}-${BUILD_NUMBER}"
      }
      steps{
        echo "Copy image to Nexus Docker Registry"
        echo "sh \"skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:admin123 docker://docker-registry.default.svc.cluster.local:5000/${app_project_dev}/tasks:${devTag} docker://nexus-registry.${components_project}.svc.cluster.local:5000/tasks:${devTag}"
      }
    
      //openshiftTag alias: 'false', destStream: 'tasks', destTag: prodTag, destinationNamespace: "$app_project_dev", namespace: "$app_project_dev", srcStream: 'tasks', srcTag: devTag, verbose: 'false'
    }
  
    // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
      environment {
        version = getVersionFromPom("pom.xml")
        devTag  = "${version}-${BUILD_NUMBER}"
      }
      steps{
        echo "Deploying container image to Development Project"
        // Update the Image on the Development Deployment Config
        echo "sh \"oc set image dc/tasks tasks=docker-registry.default.svc:5000/${app_project_dev}/tasks:${devTag} -n ${app_project_dev}"
  
      // Update the Config Map which contains the users for the Tasks application
        echo "sh \"oc delete configmap tasks-config -n $app_project_dev --ignore-not-found=true"
        echo "sh \"oc create configmap tasks-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n ${app_project_dev}"
  
      // Deploy the development application.
        //openshiftDeploy depCfg: 'tasks', namespace: "$app_project_dev", verbose: 'false', waitTime: '', waitUnit: 'sec'"
        //openshiftVerifyDeployment depCfg: 'tasks', namespace: "$app_project_dev", replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
        //openshiftVerifyService namespace: "$app_project_dev", svcName: 'tasks', verbose: 'false'
      }
    }
  
    // Run Integration Tests in the Development Environment.
    stage('Integration Tests') {
      steps{
        echo "Running Integration Tests"
        //sleep 30
  
        // Create a new task called "integration_test_1"
        echo "Creating task in the app"
        echo "sh \"curl -i -f -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${app_project_dev}.svc:8080/ws/tasks/integration_test_1"
  
        // Retrieve task with id "1"
        echo "Retrieving tasks"
        echo "sh \"curl -i -f -u 'tasks:redhat1' -H 'Content-Length: 0' -X GET http://tasks.${app_project_dev}.svc:8080/ws/tasks/1"
  
        // Delete task with id "1"
        echo "Deleting tasks"
        echo "sh \"curl -i -f -u 'tasks:redhat1' -H 'Content-Length: 0' -X DELETE http://tasks.${app_project_dev}.svc:8080/ws/tasks/1"
      }
    }

    stage('Blue/Green Deployment into Production') {
      environment {
        version = getVersionFromPom("pom.xml")
        devTag  = "${version}-${BUILD_NUMBER}"
        // The newly built version is not activated yet:
        destApp   = "tasks-green"
        activeApp = "tasks-green"//just to test out
        //activeApp = sh(returnStdout: true, script: "oc get route tasks -n ${app_project_prod} -o jsonpath='{ .spec.to.name }'").trim() // Got called, which is promissing
      }
      steps{
        // Replace xyz-tasks-dev and xyz-tasks-prod with
        // your project names
        //script{maybe//activeApp = sh(returnStdout: true, script: "oc get route tasks -n ${app_project_prod} -o jsonpath='{ .spec.to.name }'").trim()
        script{
          if (activeApp == "tasks-green") {
            destApp = "tasks-blue"
          }
          echo "Active Application:      " + activeApp
          echo "Destination Application: " + destApp
        }//script  

        // Update the Image on the Production Deployment Config
        //sh "oc set image dc/${destApp} ${destApp}=docker-registry.default.svc:5000/${app_project_dev}/tasks:${prodTag} -n ${app_project_prod}"
        
        // Update the Config Map which contains the users for the Tasks application
        //sh "oc delete configmap ${destApp}-config -n ${app_project_prod} --ignore-not-found=true"
        //sh "oc create configmap ${destApp}-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n ${app_project_prod}"
        
        // Deploy the inactive application.
        // Replace xyz-tasks-prod with the name of your production project
        //openshiftDeploy depCfg: destApp, namespace: "${app_project_prod}", verbose: 'false', waitTime: '', waitUnit: 'sec'
        //openshiftVerifyDeployment depCfg: destApp, namespace: "${app_project_prod}", replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
        //openshiftVerifyService namespace: "${app_project_prod}", svcName: destApp, verbose: 'false'
      }
    }
    
    // Switch stage (user input):
    stage('Switch over to new Version') {
      steps{
        input "Switch Production?"
        echo "Switching Production application to ${destApp}."
    
        // Replace xyz-tasks-prod with the name of your production project
        echo """sh \'oc patch route tasks -n ' + app_project_prod + ' -p \'{"spec":{"to":{"name":"' + destApp + '"}}}\''"""
      }
    }
  }//stages
}//pipeline

