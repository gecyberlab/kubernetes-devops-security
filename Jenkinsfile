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
	
	stage('Sonarqube - SAST') {
		steps {
			sh "mvn clean verify sonar:sonar -Dsonar.projectKey=numeric-application2 -Dsonar.host.url=http://devsecops-tal.eastus.cloudapp.azure.com:9000 -Dsonar.login=ac2bcbbef0432c4dfa8c1df599c3f720207a4214"
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
