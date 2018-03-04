---
layout: post
title:  "Jenkins, Docker, and sharing data between stages"
date:   2018-03-04 10:18:00 +0100
---
I've recently started moving a number of projects from their classic Jenkins Maven Project setup to Jenkins Pipeline
setups. I really like the concept of pipelines, but one obstacle I encountered was the need to specify tools for your
builds, which depend on the naming by the Jenkins administrator, and places a burden on the Jenkins administrator
to keep the tools up-to-date.

Fortunately for me, the servers in question also had Docker installed and configured for use by Jenkins, so I
could simply specify I wanted to run the build inside a Docker container:

```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.5-jdk-8'
            label 'docker'
        }
    }
}
```
There are two caveats to doing this, however:

* The only thing shared between the Jenkins instance (which can be a build slave) and the docker container is the workspace
directory. This means that the local Maven repository for the build is blank, and all dependencies will need
to be downloaded as part of the build. Depending on your use case, this might not be ideal, but there are ways around this
* Each stage you specify gets a fresh instance of the Docker container, so anything you modify inside it that is not part
of a mounted volume gets reset between stages.

This second caveat is something I ran into in a number of builds.
<!--more-->
## Example 1: Deploy config

Quite a few of the projects I work on are libraries, and need to be deployed to a private Maven repository (Artifactory
 or Nexus, depending on the project). The setup I was using was something like this:
 
 ```groovy
 pipeline {
     agent {
         docker {
             image 'maven:3.5-jdk-8'
             label 'docker'
             args '-u root'
         }
     }
     
     stage('Configure') {
       steps {
            withCredentials([usernamePassword(credentialsId: 'maven-deploy-creds', usernameVariable: 'nexusUsername', passwordVariable: 'nexusPassword')]) {
                            sh 'echo "<settings><servers><server><id>release</id><username>' +
                                    nexusUsername + '</username><password>' + nexusPassword + '</password></server></servers></settings>" > /root/.m2/settings.xml'
                        }
       }
     }
     stage('Compile') {
       steps {
         sh 'mvn clean package'
       }
     }
     stage('Deploy') {
       steps {
         sh 'mvn deploy -DskipTests=true'
       }
     }
 }
 ```
 The result? A [401 error](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/401) error on the last stage. At
 first I thought I might have gotten the credentials wrong, but then I realized what I'd done: I'd written to an unmounted
 file inside the Docker container, which got wiped between stages. To solve this I eliminated the Configure
 stage and moved the credential creation into the Deploy stage.
 
```groovy
pipeline {
  agent {
      docker {
          image 'maven:3.5-jdk-8'
          label 'docker'
          args '-u root'
      }
  }
  
  stage('Compile') {
    steps {
      sh 'mvn clean package'
    }
  }
  stage('Deploy') {
    steps {
     withCredentials([usernamePassword(credentialsId: 'maven-deploy-creds', usernameVariable: 'nexusUsername', passwordVariable: 'nexusPassword')]) {
                     sh 'echo "<settings><servers><server><id>release</id><username>' +
                             nexusUsername + '</username><password>' + nexusPassword + '</password></server></servers></settings>" > /root/.m2/settings.xml'
                 }
      sh 'mvn deploy -DskipTests=true'
    }
  }
}
```

## Example 2: Build inside container, deploy outside

Another project I worked on yielded a Docker image, and had to be deployed to a Docker registry rather than
a Maven repository. The setup I had was as follows:

```groovy
pipeline {
  agent none
  
  stage('Compile') {
    agent  {
        docker {
            image 'maven:3.5-jdk-8'
            label 'docker'
            args '-u root'
        }
    }
    
    steps {
      sh 'mvn clean package'
    }
  }
  stage('Deploy') {
    agent {
      label 'docker'
    }
    
    steps {
      sh 'docker pull my.private.registry/parent-image'
      sh 'docker build -t my-image:latest .'
      sh 'docker push my.private.registry/my-image:latest'
    }
  }
}
```

With the following Dockerfile:
```docker
FROM my.private.registry/parent-image
COPY target/my-project.war $JETTY_BASE/webapps/root.war
```

Seems pretty straightforward, doesn't it? There is one problem with this setup, however: there is no guarantee
that both stages will be executed on the same machine, and even if they are, they will not share
the same workspace folder. In other words: the build output from the first stage will not be present in the
second stage.

Jenkins provides a rather simple solution for this, with the `stage` and `unstage` steps. These are not
meant for large files however, so this isn't the best solution:

```groovy
pipeline {
  agent none
  
  stage('Compile') {
    agent  {
        docker {
            image 'maven:3.5-jdk-8'
            label 'docker'
            args '-u root'
        }
    }
    
    steps {
      sh 'mvn clean package'
      stash name: 'war', includes: '**/*.war'
    }
  }
  stage('Deploy') {
    agent {
      label 'docker'
    }
    
    steps {
      sh 'docker pull my.private.registry/parent-image'
      unstash 'war'
      sh 'docker build -t my-image:latest .'
      sh 'docker push my.private.registry/my-image:latest'
    }
  }
}
```

## Conclusion
Jenkins pipelines and Docker images work very well together, but there are a number of subtleties one has to keep in mind
while developing. That said, I will continue to deploy Jenkins pipelines wherever I can.