pipeline {
	agent {
		docker {
			image 'registry.jeroensteenbeeke.nl/jekyll-builder:latest'
			label 'docker'
			args '-u root'
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
				    sh 'mkdir -p $HOME/.ssh'
		                    sh 'cp '+ sshKeyFile +' $HOME/.ssh/id_rsa'
                		    sh 'echo host 23.94.72.113 > $HOME/.ssh/config'
		                    sh 'echo "\tStrictHostKeyChecking no" >> $HOME/.ssh/config'
		                    sh 'chmod 0400 $HOME/.ssh/config'
				    sh 'rsync --delete -r _site/ deploy@23.94.72.113:/home/deploy/_site/'
				}

			}
		}
	}
}
