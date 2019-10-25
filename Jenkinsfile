  
/* 
* Copyright (c) 2017 and Confidential to Pegasystems Inc. All rights reserved.  
*/ 

pipeline {
    agent any

    options {
      timestamps()
      timeout(time: 15, unit: 'MINUTES')
      withCredentials([
        usernamePassword(credentialsId: 'artifactory', 
            passwordVariable: 'ARTIFACTORY_PASSWORD', 
            usernameVariable: 'ARTIFACTORY_USER'),
        usernamePassword(credentialsId: 'imsadmin', 
            passwordVariable: 'IMS_PASSWORD', 
            usernameVariable: 'IMS_USER')
        ])
    }
	

    stages {
	    
	    	stage('TEST'){
            steps {
                echo 'TEST TEST TEST TEST TEST TEST TEST TEST TEST'
                sh "./gradlew exportFromArtifactory -PtargetURL=${PEGA_DEV} -Pbranch=${branchName} -PpegaUsername=${IMS_USER} -PpegaPassword=${IMS_PASSWORD}"
            }
         }
	stage('TEST DOIS'){
            steps {
                echo 'TEST TEST TEST TEST TEST TEST TEST TEST TEST DOIS'
                sh "./gradlew importOperation -PtargetURL=${PEGA_DEV} -PpegaUsername=${IMS_USER} -PpegaPassword=${IMS_PASSWORD} -Pbranch=${branchName}"
            }
         }

        stage('Check for merge conflicts'){
            steps {
                echo ('Clear workspace')
                dir ('build/export') {
                    deleteDir()
                }

                echo 'Determine Conflicts'
                sh "./gradlew getConflicts -PtargetURL=${PEGA_DEV} -Pbranch=${branchName} -PpegaUsername=${IMS_USER} -PpegaPassword=${IMS_PASSWORD}"
            }
        }

        stage('Run unit tests'){
          steps {
            echo 'Execute tests'

            withEnv(['TESTRESULTSFILE="TestResult.xml"']) {
              sh "./gradlew executePegaUnitTests -PtargetURL=${PEGA_DEV} -PpegaUsername=${IMS_USER} -PpegaPassword=${IMS_PASSWORD} -PtestResultLocation=${WORKSPACE} -PtestResultFile=${TESTRESULTSFILE}"
                    
             // junit(allowEmptyResults: true, testResults: "${env.WORKSPACE}/${env.TESTRESULTSFILE}")

              script {
                if (currentBuild.result != null) {
                  input(message: 'Ready to share tests have failed, would you like to abort the pipeline?')
                }
              }
            }
          }
       }

       stage('Merge branch'){
        when {
          environment name: "PERFORM_MERGE", value: "true"
        }

        steps{

            echo 'Perform Merge' 

            sh "./gradlew merge -PtargetURL=${env.PEGA_DEV} -Pbranch=${branchName} -PpegaUsername=${IMS_USER} -PpegaPassword=${IMS_PASSWORD}"
            echo 'Evaluating merge Id from gradle script = ' + env.MERGE_ID
            timeout(time: 5, unit: 'MINUTES') {
                echo "Setting the timeout for 1 min.."
                retry(10) {
                    echo "Merge is still being performed. Retrying..."
                    sh "./gradlew getMergeStatus -PtargetURL=${env.PEGA_DEV} -PpegaUsername=${IMS_USER} -PpegaPassword=${IMS_PASSWORD}"
                    echo "Merge Status : ${env.MERGE_STATUS}"
                }
            }
          }
        }
	    
	    stage('Publish to Artifactory') {

            steps {
                echo 'Publishing to Artifactory '
                sh "./gradlew artifactoryPublish -PartifactoryUser=${ARTIFACTORY_USER} -PartifactoryPassword=${ARTIFACTORY_PASSWORD}"
            }
        }


        stage('Regression Tests') {

            steps {
                echo 'Run regression tests'
                echo 'Publish to production repository'
            }
        }

        stage('Create restore point') {

            steps {
                echo 'Creating restore point'
                sh "./gradlew createRestorePoint -PtargetURL=${PEGA_PROD} -PpegaUsername=${IMS_USER} -PpegaPassword=${IMS_PASSWORD}"
            }
        }

  }

  post {
    failure {
      mail (
          subject: "${JOB_NAME} ${BUILD_NUMBER} merging branch ${branchName} has failed",
          body: "Your build ${env.BUILD_NUMBER} has failed.  Find details at ${env.RUN_DISPLAY_URL}", 
          to: notificationSendToID
      )
    }   
  }
}
