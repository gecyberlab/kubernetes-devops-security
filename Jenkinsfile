pipeline {
  agent any
  stages {
  	stage('Build Artifact') {
  		steps {
  			sh "mvn clean package -DskipTests=true"
  			archive 'target/*.jar'
  		}
	}
	stage('Unit Test') {
		steps {
			sh "mvn test"
		}
	}
	
	stage('Mutation Tests - PIT') {
      steps {
        sh "mvn org.pitest:pitest-maven:mutationCoverage"
      }
    }

	stage('SonarQube - SAST') {
      steps {
        withSonarQubeEnv('sonarqube-server') {
          sh "mvn sonar:sonar -Dsonar.projectKey=numeric -Dsonar.host.url=http://devsecops-tal.eastus.cloudapp.azure.com:9000"
        }
        timeout(time: 3, unit: 'MINUTES') {
          script {
            waitForQualityGate abortPipeline: true
          }
        }
      }
    }
    
      stage('Vulnerability Scan - Docker') {
      steps {
        parallel(
          "Dependency Scan": {
            sh "mvn dependency-check:check"
          },
          "Trivy Scan": {
            sh "bash trivy-docker-image-scan.sh"
          }
        )
      }
    }

    
  
	stage('Docker build and push') {
		steps {
			withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
			sh 'printenv'
			sh 'sudo docker build -t talmanor/numeric-app:""$GIT_COMMIT"" .'
			sh 'docker push talmanor/numeric-app:""$GIT_COMMIT""'
			}
		}
	}
	stage('Kubernetes deployment - DEV') {
		steps {
			withKubeConfig([credentialsId: "kubeconfig"]) {
			sh "sed -i 's#replace#talmanor/numeric-app:$GIT_COMMIT#g' k8s_deployment_service.yaml"
			sh "kubectl apply -f k8s_deployment_service.yaml"
			}
		}
	}
}
		post {
			always {
				junit 'target/surefire-reports/*.xml'
				jacoco execPattern: 'target/jacococ.exec'
		        dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
	            pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
			}
		}

}
