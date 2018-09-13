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

        stage('Run multi-productionconfig and Environment Create') {
            parallel {
                stage('Run multi-productionconfig') {
                    steps {
                        script {
                            dir("${WORKSPACE}/ingest"){
                                sh "${WORKSPACE}/config/jenkins/bin/ingest"
                            }
                        }
                    }
                }

                stage('Create Ephemeral Environment') {
                    steps {
                        script {
                            if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                                //env.SRCDIR = "${WORKSPACE}/promote"
                                echo "${TARGET}"
                                sh "cd ${WORKSPACE}/tools/terraform/hybris-test"
                                echo "============================ Environment Creation START      ==============================="
                                dir("${TERRFORM_DIR}"){
                                  echo "${DRAFT_BRANCH_NAME}"
                                  sh "pwd"
                                  sh "${WORKSPACE}/config/jenkins/bin/update_state.sh ${DRAFT_BRANCH_NAME} main.tf"
                                  sh "terraform init -input=false"
                                  sh "terraform apply -var instance_count=1 -auto-approve -var build_name=${TARGET}"
                                }

                                env.DB = sh(returnStdout: true, script: "cd $TERRFORM_DIR; terraform output db_instance_endpoint").trim()
                                sh "${WORKSPACE}/config/jenkins/bin/make-deploytarget-single.sh ${TARGET}"
                                echo "============================ Environment Creation COMPLETE   ==============================="
                            } else {
                              echo "IN ELSE: RESULT: ${currentBuild.result}"
                              currentBuild.result = 'FAILURE'
                            }
                        }
                    }
                }
            }
        }

        stage('Wait for Ephemeral Environment'){
            steps {
                timeout(10){
                    sh "${WORKSPACE}/config/jenkins/bin/verify_ssh_connection.sh ${TARGET}.dev-newlook.com"
                }
            }
        }

        stage('Deploy') {
            steps {
                echo "Running config/jenkins/bin/deployWrapper ${TARGET}"
                sh "sh ${WORKSPACE}/config/jenkins/bin/deployWrapper ${TARGET}"
                copyArtifacts filter: 'tools/newlook-simulator/newlook-simulator-0.0.1-SNAPSHOT.jar', projectName: 'Salmon/neo-simulator-build', selector: lastSuccessful()
                sh "${WORKSPACE}/config/jenkins/bin/deploySimulator ${TARGET} newlook-simulator-0.0.1-SNAPSHOT.jar"
            }
        }

        stage('Wait for hybris to start up'){
            steps {
                timeout(30){
                    sh "${WORKSPACE}/config/jenkins/bin/verify_storefront.sh ${TARGET}.dev-newlook.com"
                    echo "Storefront is up and running"
                }
            }
        }

        stage('Testing') {
            parallel {
                stage('Storefront Test'){
                    steps {
                        /*timeout(30){
                          sh "${WORKSPACE}/config/jenkins/bin/verify_storefront.sh ${env.TARGET}.dev-newlook.com"
                          echo "Storefront is up and running"
                          echo "Going to sleep for 10 mins for debug purpose"
                          sleep 600
                          echo "after sleep(secs)"
                        }*/
                        echo "TBD"
                    }
                }

                stage('CIT tests') {
                  when {
                    expression {
                      return DO_CIT_TESTS == "true";
                    }
                  }
                    steps {
                        sh "mkdir -p ${WORKSPACE}/build-reports/bddfeatures"
                        sh "echo 'Requested BDD Synchronized test results'"
                        sh "curl 'http://${TARGET}.dev-newlook.com:9001/neoworksCucumberServer/features/bddfeatures/run?format=json&threads=1' -o build-reports/bddfeatures/results.json"
                        sh "echo 'Requested BDD Parallel test results'"
                        sh 'curl "http://${TARGET}.dev-newlook.com:9001/neoworksCucumberServer/features/bddfeatures/run?format=json&threads=10&tags=~@Parallel" -o build-reports/bddfeatures/parallel.json'
                        cucumber buildStatus: 'FAILURE', failedFeaturesNumber: 1, failedScenariosNumber: 1, failedStepsNumber: 1, fileIncludePattern: '*.json', jsonReportDirectory: "${WORKSPACE}/build-reports/bddfeatures", pendingStepsNumber: 1, skippedStepsNumber: 1, sortingMethod: 'ALPHABETICAL', undefinedStepsNumber: 1
                    }

                }

                stage('Rest assured tests') {
                  when {
                    expression {
                      return DO_REST_ASSURED_TESTS == "true";
                    }
                  }
                    steps {
                        sh "mvn -f ${WORKSPACE}/test/API/pom.xml clean install test -PhybrisTestEnv -DhybrisBaseUrl=https://${TARGET}.dev-newlook.com:4444/ -DhybrisBaseWebsite.url=http://${TARGET}.dev-newlook.com -DhybrisResourcePath=hybris-demo-ds"
                    }

                }

                stage('Other tests') {
                    steps {
                        echo "TBD"
                    }
                }
            }
        }

        stage('Bless Artifact & Ephemeral Env - Destroy') {
            steps {
                script {
                    if (currentBuild.result == null || currentBuild.result == 'SUCCESS'){
                        echo "Bless Artifact - Start"
                        sh "ssh -o StrictHostKeyChecking=no nexus.dev-newlook.com sudo /home/jenkins/bin/bless_draft_branch.sh ${DRAFT_BRANCH_NAME} ${env.DRAFT_BUILD_NUMBER}"
                        echo "Artifact blessed"
                        echo "============================ Environment Destroy START      ==============================="
                        input message: 'Approve Destroy?', ok: 'Apply'
                        sh "cd $TERRFORM_DIR; terraform destroy -auto-approve -var build_name=${TARGET}"
                        echo "============================ Environment Creation COMPLETE   ==============================="
                    } else {
                        echo "IN ELSE: RESULT: ${currentBuild.result}"
                        currentBuild.result = 'FAILURE'
                        try {
                            timeout(time: 18, unit: 'HOURS') {
                                input message: 'Approve Destroy?', ok: 'Apply'
                                echo "Build is failed. Waiting for user input... If there is no input, after 18 hours,  will destroy ephemeral env"
                                sh "cd ${TERRFORM_DIR}; terraform destroy -auto-approve -var build_name=${TARGET}"
                            }
                        } catch (err) {
                            echo "No user input provided for 18 hours, so ephemeral is going to be destroyed"
                            sh "cd ${TERRFORM_DIR}; terraform destroy -var instance_count=1 -auto-approve -var build_name=${TARGET}"
                        }
                    }
                }
            }
        }
    }
    post {
        success { slackSend color: 'good', message: "SUCCESS: Job ${env.slackMessage}" }
        failure { slackSend color: 'danger', message: "FAILED: Job ${env.slackMessage}" }
        aborted { slackSend color: '#D3D3D3', message: "IT'S NOT ME, IT'S YOU: Aborted Job ${env.slackMessage}"}
    }
}
