// Set your project Prefix using your GUID
def prefix      = "rn9qm"

// Set variable globally to be available in all stages
// Set Maven command to always include Nexus Settings
def mvnCmd      = "mvn -s ./nexus_openshift_settings.xml"
// Set Development and Production Project Names
def devProject  = "${prefix}-tasks-dev"
def prodProject = "${prefix}-tasks-prod"
// Set the tag for the development image: version + build number
def devTag      = "0.0-0"
// Set the tag for the production image: version
def prodTag     = "0.0"
def destApp     = "tasks-green"
def activeApp   = ""

pipeline {
  agent {
    // Using the Jenkins Agent Pod that we defined earlier
    label "maven-appdev"
  }
  stages {
    // Checkout Source Code and calculate Version Numbers and Tags
    stage('Checkout Source') {
      steps {
        // TBD: Get code from protected Git repository
        git credentialsId: '993494bc-1a96-4212-9f9e-458c07208366', url: 'https://gogs-gogs-rn9qm-gogs.apps.cluster-70c3.70c3.sandbox1862.opentlc.com/CICDLabs/openshift-tasks-private'
        //checkoud
        checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '993494bc-1a96-4212-9f9e-458c07208366', url: 'https://gogs-gogs-rn9qm-gogs.apps.cluster-70c3.70c3.sandbox1862.opentlc.com/CICDLabs/openshift-tasks-private']]])
       script {
          def pom = readMavenPom file: 'pom.xml'
          def version = pom.version
          
          // Set the tag for the development image: version + build number
          devTag  = "${version}-" + currentBuild.number
          // Set the tag for the production image: version
          prodTag = "${version}"

          // TBD: Set the tag for the development image: version + build number.
          // Example: def devTag  = "0.0-0"

          // TBD: Set the tag for the production image: version
          // Example: def prodTag = "0.0"

        }
      }
    }

    // Using Maven build the war file
    // Do not run tests in this step
    stage('Build War File') {
      steps {
        echo "Building dev-tag version ${devTag} and prod-tag version ${prodTag}"
        sh "${mvnCmd} clean package -DskipTests=true"

        // TBD

      }
    }

    // Using Maven run the unit tests
    stage('Unit Tests') {
      steps {
        echo "Running Unit Tests"
        sh "${mvnCmd} test"
        
        // This next step is optional.
        // It displays the results of tests in the Jenkins Task Overview
        step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
        }
    }

    //Using Maven call SonarQube for Code Analysis
    stage('Code Analysis') {
      steps {
        echo "Running Code Analysis"
        // orig sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-${prefix}-sonarqube.apps.fab.example.opentlc.com/ -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"
        sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-rn9qm-sonarqube.apps.cluster-70c3.70c3.sandbox1862.opentlc.com/ -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"

        //http://sonarqube-rn9qm-sonarqube.apps.cluster-70c3.70c3.sandbox1862.opentlc.com/
      }
    }

    // Publish the built war file to Nexus
    stage('Publish to Nexus') {
      steps {
        echo "Publish to Nexus"
        //sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus.${prefix}-nexus.svc.cluster.local:8081/repository/releases"
        // TBD
        // http://nexus-rn9qm-nexus.apps.cluster-70c3.70c3.sandbox1862.opentlc.com/#browse/browse:maven-all-public
        sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus-rn9qm-nexus.apps.cluster-70c3.70c3.sandbox1862.opentlc.com/repository/releases/"
      }
    }

    // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      steps 
      {
        echo "Building OpenShift container image tasks:${devTag}"
        script 
        {
            openshift.withCluster() {
                openshift.withProject("${devProject}") {
                    openshift.selector("bc", "tasks").startBuild("--from-file=./target/openshift-tasks.war", "--wait=true") 
            openshift.tag("tasks:latest", "tasks:${devTag}")        
        }
      }
    }
      }
    }

    // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
        steps {
            echo "Deploying container image to Development Project"
                script {
                  // Update the Image on the Development Deployment Config
                  openshift.withCluster() {
                    openshift.withProject("${devProject}") {
                      // OpenShift 4
                      openshift.set("image", "dc/tasks", "tasks=image-registry.openshift-image-registry.svc:5000/${devProject}/tasks:${devTag}")

                      // Recreate the Config Map which contains the users for the Tasks application
                      // (just in case the properties files changed in the latest commit)
                      openshift.selector('configmap', 'tasks-config').delete()
                      def configmap = openshift.create('configmap', 'tasks-config', '--from-file=./configuration/application-users.properties', '--from-file=./configuration/application-roles.properties' )
            
                      // Deploy the development application.
                      openshift.selector("dc", "tasks").rollout().latest();
            
                      // Wait for application to be deployed
                      sleep 5
                      def dc = openshift.selector("dc", "tasks").object()
                      def dc_version = dc.status.latestVersion
                      def rc = openshift.selector("rc", "tasks-${dc_version}").object()
            
                      echo "Waiting for ReplicationController tasks-${dc_version} to be ready"
                      while (rc.spec.replicas != rc.status.readyReplicas) {
                        sleep 5
                        rc = openshift.selector("rc", "tasks-${dc_version}").object()
                      }
        }
      }
    }
  }
}
      

// Run Integration Tests in the Development Environment.
stage('Integration Tests') {
  steps {
    echo "Running Integration Tests"
    script {
      def status = "000"

      // Create a new task called "integration_test_1"
      echo "Creating task"
      status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/integration_test_1").trim()
      echo "Status: " + status
      if (status != "201") {
          error 'Integration Create Test Failed!'
      }

      echo "Retrieving tasks"
      status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Accept: application/json' -X GET http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
      if (status != "200") {
          error 'Integration Get Test Failed!'
      }

      echo "Deleting tasks"
      status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -X DELETE http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
      if (status != "204") {
          error 'Integration Create Test Failed!'
      }
    }
  }
}
    
    
// Copy Image to Nexus Container Registry
stage('Copy Image to Nexus Container Registry') {
  steps {
    echo "Copy image to Nexus Container Registry"
    script {
      sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:app_deploy docker://image-registry.openshift-image-registry.svc.cluster.local:5000/${devProject}/tasks:${devTag} docker://nexus-registry.${prefix}-nexus.svc.cluster.local:5000/tasks:${devTag}"

      // Tag the built image with the production tag.
      openshift.withCluster() {
        openshift.withProject("${prodProject}") {
          openshift.tag("${devProject}/tasks:${devTag}", "${devProject}/tasks:${prodTag}")
        }
      }
    }
  }
}
    

    // Blue/Green Deployment into Production
// -------------------------------------
// Do not activate the new version yet.
stage('Blue/Green Production Deployment') {
  steps {
    echo "Blue/Green Deployment"
    script {
      openshift.withCluster() {
        openshift.withProject("${prodProject}") {
          activeApp = openshift.selector("route", "tasks").object().spec.to.name
          if (activeApp == "tasks-green") {
            destApp = "tasks-blue"
          }
          echo "Active Application:      " + activeApp
          echo "Destination Application: " + destApp

          // Update the Image on the Production Deployment Config
          def dc = openshift.selector("dc/${destApp}").object()

          dc.spec.template.spec.containers[0].image="image-registry.openshift-image-registry.svc:5000/${devProject}/tasks:${prodTag}"

          openshift.apply(dc)

          // Recreate Config Map in change config files changed in the source
          openshift.selector("configmap", "${destApp}-config").delete()
          def configmap = openshift.create("configmap", "${destApp}-config", "--from-file=./configuration/application-users.properties", "--from-file=./configuration/application-roles.properties" )

          // Deploy the inactive application.
          openshift.selector("dc", "${destApp}").rollout().latest();

          // Wait for application to be deployed
          sleep 5
          def dc_prod = openshift.selector("dc", "${destApp}").object()
          def dc_version = dc_prod.status.latestVersion
          def rc_prod = openshift.selector("rc", "${destApp}-${dc_version}").object()
          echo "Waiting for ${destApp} to be ready"
          while (rc_prod.spec.replicas != rc_prod.status.readyReplicas) {
            sleep 5
            rc_prod = openshift.selector("rc", "${destApp}-${dc_version}").object()
          }
        }
      }
    }
  }
}

stage('Switch over to new Version') 
{
  steps {
            input "Switch Production?"

            echo "Switching Production application to ${destApp}."
            script 
            {
                openshift.withCluster() 
                {
                    openshift.withProject("${prodProject}") 
                    {
                        def route = openshift.selector("route/tasks").object()
                        route.spec.to.name="${destApp}"
                        openshift.apply(route)
                    }
                }
            }
        }
}

}}
