#!groovy

pipeline {
    //agent any
    agent { label 'linux' }
    

    environment {
      DRAFT_BRANCH_NAME=      "${env.DRAFT_BRANCH_NAME}".toLowerCase()
      slackMessage =          "${env.JOB_NAME}, build ${env.BUILD_NUMBER}\n${env.BUILD_URL}\nBranch: ${DRAFT_BRANCH_NAME}-${env.DRAFT_BUILD_NUMBER}"
      AWS_PROFILE =           "newlook-nonprod-ddt1"
      TARGET =                "qa-cicd-${DRAFT_BRANCH_NAME}-${env.DRAFT_BUILD_NUMBER}"
      SRCDIR =                "${WORKSPACE}/promote"
      TERRFORM_DIR =          "${WORKSPACE}/tools/terraform/cicd-ephemeral"
      DO_CIT_TESTS =          "${env.DO_CIT_TESTS}"
      DO_REST_ASSURED_TESTS = "${env.DO_REST_ASSURED_TESTS}"

    }

    stages {
        stage('Download Artifacts') {
            steps {
                script {
                    slackSend (color: '#FFFF00', message: "STARTED: Job '${env.slackMessage}' ")
                    echo "Node: ${env.NODE_NAME}"
                    echo "Job status before ingest: ${currentBuild.result}"
                    echo "DRAFT_BUILD_NUMBER: ${DRAFT_BUILD_NUMBER}"
                    if (currentBuild.result == null || currentBuild.result == 'SUCCESS'){
                        if (fileExists("${WORKSPACE}/ingest")) {
                            sh 'rm -rf ${WORKSPACE}/ingest'
                        }
                        sh "mkdir ${WORKSPACE}/ingest"
                        if (fileExists("${WORKSPACE}/promote")) {
                            sh 'rm -rf ${WORKSPACE}/promote'
                        }
                        sh "mkdir ${SRCDIR}"
                        dir("${SRCDIR}"){
                            echo "Download artifact from Nexus - START"
                            sh "wget -q ${env.ARTIFACT_URL}"
                            echo "Download artifact from Nexus - COMPLETE"
                            sh "ls ${SRCDIR}"
                            sh "tar -xvf NewLook-hybris-*.tar.gz"
                        }
                    } else {
                            echo "IN ELSE: RESULT: ${currentBuild.result}"
                            currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
   
 }
