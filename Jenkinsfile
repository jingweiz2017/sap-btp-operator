#!/usr/bin/env groovy

library identifier: 'steward-pipeline-utils@v0.47.2',
    retriever: modernSCM([$class: 'GitSCMSource',
        remote: 'https://github.wdf.sap.corp/steward-ci/steward-pipeline-utils.git',
        credentialsId: 'pccibuild_private_github_wdf_user_token'
    ])

def MILESTONE = 0

pipeline {
    agent any
    parameters{
        booleanParam(name: 'executeWithMalwareScan', description: '', defaultValue: true)
    }
    options {
        skipDefaultCheckout(true)
        timestamps()
    }
    stages {
        stage('Initialize') {
            steps {
                deleteDir()
                script {
                    def scmInfo = checkout scm
                    stewardUtils.loadPiperLibraries(script: this)
                    setupCommonPipelineEnvironment script: this, scmInfo: scmInfo
                }
            }
        }
        stage('Central Xmake Build (Stage)') {
            when {
                branch 'master'
            }
            steps {
                milestoneLock(ordinal: ++MILESTONE) {
                    executeBuild script: this
                    stash name: 'dockerMetadata', includes: 'docker.metadata.json,xmake_stage.json'
                }
            }
        }
        stage('Malware Scan') {
            agent {
                label 'docker'
            }
            when {
                branch 'master'
                expression {
                    return params.executeWithMalwareScan
                }
            }
            steps {
                scanDockerImageForMalware script: this,
                    dockerMetadataStashName: 'dockerMetadata',
                    extractCompressedPackages: true
            }
        }
        stage('Manual approval') {
            agent none
            when {
                branch 'master'
                beforeInput true
            }
            options {
                timeout(time: 1, unit: 'DAYS')
            }
            steps {
                emailext(
                    mimeType: 'text/html',
                    subject: "[${commonPipelineEnvironment.githubRepo}] Release needs APPROVAL",
                    body: "Please APPROVE or REJECT the Xmake promote build for ${commonPipelineEnvironment.githubRepo} within the next day.",
                    to: "${commonPipelineEnvironment.configuration.general.notificationRecipients}"
                )
                input 'Promote central build'
            }
        }
        stage('Central Xmake Build (Promote)') {
            when {
                branch 'master'
            }
            steps {
                milestoneLock(ordinal: ++MILESTONE) {
                    executeBuild script: this, buildType: 'xMakePromote'
                }
            }
        }
    } //stages
    post {
        always {
            mailSendNotification script: this, wrapInNode: true
        }
        cleanup {
            deleteDir()
        }
    }
} //pipeline
