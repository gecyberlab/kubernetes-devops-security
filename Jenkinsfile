pipeline {
  agent
  any
  stages {
  	stage(
  		'Build Artifact'
  	) {
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
			sh "set -i 's#replace#talmanor/numeric-app:$GITHUB_COMMIT#g' k8s_deployment_service.yaml"
			sh "kubectl apply -f k8s_deployment_service.yaml"
			}
		}
	}
}
}
