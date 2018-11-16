pipeline {
	agent {
		docker {
			image 'registry.jeroensteenbeeke.nl/jekyll-builder:latest'
			label 'docker'
			args '-u jenkins'
		}
	}

	options {
		buildDiscarder(logRotator(numToKeepStr: '5'))
		disableConcurrentBuilds()
	}

	stages {
		stage('Build') {
			steps {
				sh 'bundle update'
				sh 'bundle exec jekyll build'
			}
		}

		stage('Publish') {
			steps {
				withCredentials([sshUserPrivateKey(credentialsId: 'jekyll-deploy-key', keyFileVariable: 'sshKeyFile')]) {
				    sh 'mkdir -p /home/jenkins/.ssh'
		                    sh 'cp '+ sshKeyFile +' /home/jenkins/.ssh/id_rsa'
                		    sh 'echo host 23.94.72.113 > /home/jenkins/.ssh/config'
		                    sh 'echo "\tStrictHostKeyChecking no" >> /home/jenkins/.ssh/config'
		                    sh 'chmod 0400 /home/jenkins/.ssh/config'
				    sh 'rsync --delete -r _site/ deploy@23.94.72.113:/home/deploy/_site/'
				}

			}
		}
	}
}
