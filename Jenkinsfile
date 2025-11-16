@Library('devops-platform') _

pipeline {
	agent any
	triggers { githubPush() }

	tools {
		jdk 'JDK 21'
		maven 'Maven 3.9.9'
	}

	environment {
		REPO_OWNER = 'ChellaVigneshKP'
		REPO_NAME = 'lib-crypto'
	}

	stages {
		stage('Checkout') { steps { script {
			githubChecks.startCheck('Checkout', 'Starting pipeline')
			checkout scm
			env.GIT_COMMIT = utilities.getGitCommit()
		} } }

		stage('Version') { steps { script { versioning.determineVersion() } } }
		stage('Build') { steps { script { buildMavenApp() } } }
		stage('Test') { steps { script { runTests() } } }
		stage('Security') { steps { script { securityScan() } } }
		stage('Deploy') { steps { script { artifactoryDeploy() } } }

		stage('Tag Release') {
			when {
				expression { env.VERSION_TYPE == 'RELEASE' && utilities.isMainBranch() }
			}
			steps { script {
				utilities.executeScript('git-utils', [
					'action': 'create-tag',
					'version': env.DEPLOY_VERSION
				])
			} }
		}
	}

	post {
		always {
			script {
				githubChecks.endMainCheck(
					currentBuild.result == 'SUCCESS' ? 'success' : 'failure',
					"Build ${currentBuild.result} for ${env.DEPLOY_VERSION ?: 'unknown'}"
				)
			}
		}
	}
}