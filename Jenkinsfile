pipeline {
  agent any
  tools {
      maven 'maven3'
      jdk 'JDK8'
      jfrog 'cli'
    }
   environment {
        DOCKER_USERNAME = credentials('JFROG_USER')
	DOCKER_IMAGE_NAME = "$DOCKER_REGISTRY/$DOCKER_REPO/webapp.${env.BUILD_NUMBER}"
        ARTIFACTORY_ACCESS_TOKEN = credentials('jf_access_token')
        WEBHOOK_URL = credentials("webhook_url")
        BUILD_NO = "${env.BUILD_NUMBER}"
	DOCKER_REGISTRY = 'fisdemo1.jfrog.io'
        DOCKER_REPO = 'fis-demo-dockerhub-docker-local'
        DOCKER_PASSWORD = credentials('JFROG_PASSWORD')
	DOCKER_TAG = 'latest'
    }
    stages {    
        stage('Build maven ') {
            steps { 
                    sh 'pwd'      
                    sh 'mvn  clean install package'
            }
        }
        
        stage('Copy Artifact') {
           steps { 
                   sh 'pwd'
		   sh 'cp -r target/*.war docker'
           }
        }
	stage('Checkstyle Analysis'){
            steps {
                    sh 'mvn checkstyle:checkstyle'
            }
        }
        stage('Sonar Analysis') {
            environment {
                scannerHome = tool 'sonar4.7'
            }
            steps {
               withSonarQubeEnv('sonar') {
                   sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=webapp \
                   -Dsonar.projectName=webapp \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
		   -Dsonar.java.binaries=target/main/webapp/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
              }
            }
        }
        stage('Build docker image') {
           steps {
               script {      
                 def customImage = docker.build("$DOCKER_IMAGE_NAME", "./")                   
           }
        }
	}
	stage('Trivy Scan') {
            steps {
               sh 'trivy image $DOCKER_IMAGE_NAME  --output report.html || true'
	  }
        }
	stage('Push image to JFROG') {
	   steps {
	      sh "docker login -u $DOCKER_USERNAME -p '$DOCKER_PASSWORD' $DOCKER_REGISTRY"
	      sh "docker tag $DOCKER_IMAGE_NAME $DOCKER_IMAGE_NAME:$DOCKER_TAG"
              sh "docker push $DOCKER_IMAGE_NAME:$DOCKER_TAG"
			}
		}
	stage('Push image to Dockerhub') {
	   steps {
	       withCredentials([usernameColonPassword(credentialsId: 'w_docker', variable: 'w_docker')]) {
                sh 'docker login -u ahmedwahi314 -p ${w_docker_pass}'
	        sh "docker tag $DOCKER_IMAGE_NAME ahmedwahi314/webapp.${BUILD_ID}:$DOCKER_TAG"
                sh "docker push ahmedwahi314/webapp.${BUILD_ID}:$DOCKER_TAG"
            }
		}
	}
 
    //     stage('Notification') {
    //         steps {
    //             office365ConnectorSend webhookUrl: '$WEBHOOK_URL',
    //             message: 'build is success',
    //             status: 'Success'            
    //         }
    //     }
     }
	post {
          success {
            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: '.', reportFiles: 'report.html', reportName: 'Trivy Scan', reportTitles: 'Trivy Scan', useWrapperFileDirectly: true])
            echo 'Build successfully!'   
        }

          failure {
            echo 'Build Failed'
        }
    }      
}
