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
		post {
			always {
				junit 'target/surefire-reports/*.xml'
				jacoco execPattern: 'target/jacococ.exec'
			}
		}
	}
	
	stage('SonarQube - SAST') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh "mvn sonar:sonar -Dsonar.projectKey=numeric -Dsonar.host.url=http://devsecops-tal.eastus.cloudapp.azure.com:9000"
        }
        timeout(time: 3, unit: 'MINUTES') {
          script {
            waitForQualityGate abortPipeline: true
          }
        }
      }
    }
  
	stage('Mutation Tests - PIT') {
      steps {
        sh "mvn org.pitest:pitest-maven:mutationCoverage"
      }
      post {
        always {
          pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
        }
      }
    }
	stage('Docker build and push') {
		steps {
			withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
			sh 'printenv'
			sh 'docker build -t talmanor/numeric-app:""$GIT_COMMIT"" .'
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
}
