@Library('devops-platform') _

pipeline {
	agent any
	triggers { githubPush() }

	tools {
		jdk 'JDK 21'
		maven 'Maven 3.9.9'
	}

	environment {
		ARTIFACTORY_URL = 'https://jfrog.chellavignesh.com/artifactory'
        REPO_RELEASE_ID = 'devops-jfrog-libs-release'
        REPO_SNAPSHOT_ID = 'devops-jfrog-libs-snapshot'
		MAIN_CHECK_NAME = 'Chella Jenkins CI'
	}

	options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
    }

	stages {
		stage('Checkout') {
            steps {
                script {
                    githubChecks.startCheck('Checkout', 'Fetching source code from GitHub')
                    try {
                        deleteDir()
                        checkout scm
                        env.GIT_COMMIT = utilities.getGitCommitFull()
                        githubChecks.endCheck('Checkout', 'success', '✅ Repository cloned successfully')
                    } catch (Exception e) {
                        githubChecks.endCheck('Checkout', 'failure', "❌ Checkout failed: ${e.message}")
                        throw e
                    }
                }
            }
        }

		stage('Determine Version') {
            steps {
                script {
                    versioning.determineVersion([
                        versionStrategy: 'semantic',
                        useGitTags: true
                    ])
                }
            }
        }
		stage('Build') {
            steps {
                script {
                    buildMavenApp([
                        goals: 'clean compile',
                        skipTests: false,
                        javaVersion: '21'
                    ])
                }
            }
        }
		stage('Test') {
            steps {
                script {
                    runTests([
                        testTool: 'maven',
                        coverageTool: 'jacoco',
                        failOnNoTests: true
                    ])
                }
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }
		stage('Security Scan') {
            steps {
                script {
                    securityScan([
                        scanner: 'trivy',
                        failOnVulnerability: true,
                        severityThreshold: 'HIGH,CRITICAL'
                    ])
                }
            }
        }
		stage('Deploy') {
            steps {
                script {
                    artifactoryDeploy([
                        credentialsId: 'devops-jfrog',
                        version: env.DEPLOY_VERSION
                    ])
                }
            }
        }

		stage('Release Tagging') {
            when {
                expression { env.VERSION_TYPE == 'RELEASE' && utilities.isMainBranch() }
            }
            steps {
                script {
                    githubChecks.startCheck('Release_Tagging', "Creating Git tag v${env.DEPLOY_VERSION}")
                    try {
                        withCredentials([usernamePassword(
                            credentialsId: 'github-app-jenkins',
                            usernameVariable: 'GITHUB_APP',
                            passwordVariable: 'GITHUB_ACCESS_TOKEN'
                        )]) {
                            utilities.executeScript('git/git-utils', [
                                'action': 'create-tag',
                                'version': env.DEPLOY_VERSION
                            ])
                        }
                        githubChecks.endCheck('Release_Tagging', 'success', "🏷️ Tagged release v${env.DEPLOY_VERSION}")
                    } catch (Exception e) {
                        githubChecks.endCheck('Release_Tagging', 'failure', "❌ Tagging failed: ${e.message}")
                        throw e
                    }
                }
            }
        }
	}

	post {
        success {
            script {
                wrap([$class: 'BuildUser']) {
                    def triggerUser = env.BUILD_USER ?: 'Automated Trigger'
                    githubChecks.endMainCheck('success',
                        "✅ All stages completed successfully for ${env.DEPLOY_VERSION}\n👤 Triggered by: ${triggerUser}")
                }
            }
        }
        failure {
            script {
                wrap([$class: 'BuildUser']) {
                    def triggerUser = env.BUILD_USER ?: 'Automated Trigger'
                    githubChecks.endMainCheck('failure',
                        "❌ One or more stages failed.\n👤 Triggered by: ${triggerUser}")
                }
            }
        }
    }
}
