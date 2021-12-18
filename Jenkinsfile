pipeline {
  agent any
  
    environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "talmanor/numeric-app:${GIT_COMMIT}"
    applicationURL = "http://devsecops-tal.eastus.cloudapp.azure.com"
    applicationURI = "/increment/99"
  }
  
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
          },
          "OPA onftest": {
            sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
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
	stage('OWASP ZAP - DAST') {
      steps {
        withKubeConfig([credentialsId: 'kubeconfig']) {
          sh 'bash zap.sh'
        }
      }
    }
}
		post {
			failure
			{
				kubectl rollout undo deployment/devsecops
			}
			always {
				junit 'target/surefire-reports/*.xml'
				jacoco execPattern: 'target/jacococ.exec'
		        dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
	            pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
	            publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'owasp-zap-report', reportFiles: 'zap_report.html', reportName: 'OWASP ZAP HTML Report', reportTitles: 'OWASP ZAP HTML Report'])	            
			}
			
		}

}
