pipeline {
    agent any
	
	environment {
	
		dockerRepo = "serverlessefstoolchainjava"
		AWS_ACCOUNT_ID = "962109799108"
		AWS_REGION = "eu-west-1"
		DOCKER_CACHE_IMAGE_VERSION = "latest"

		package_version = readMavenPom().getVersion()
		applicationName = readMavenPom().getArtifactId()
		groupID = readMavenPom().getGroupId()
		dockerRegistry = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
		
    }
	
    stages {
	    stage('Prepa baking') {
            steps {
                echo 'Getting previous image ...'
				sh 'echo \"Si l\'image cache n\'existe pas dans le repo ECR elle est reconstruire, sinon elle est telechargee\"'
				sh 'chmod +x build-docker.sh'
				sh './build-docker.sh $dockerRepo $DOCKER_CACHE_IMAGE_VERSION dockerfile_basis $AWS_REGION $AWS_ACCOUNT_ID'
            }
        }
        stage('Build') {
            steps {
                echo 'Building ...'
				//sh 'mvn -T 10 -Dmaven.test.skip=true clean install'
				sh 'mvn -T 1C -Dmaven.test.skip=true clean package'
            }
        }
		
		stage('Unit test') {
            steps {
                echo 'Unit testing ...'
				sh 'mvn -T 1C test'
            }
        }
/*
		stage('Publish snapshot') {
            steps {
                echo 'Publising into the snapshot repo ...'
				sh 'mvn jar:jar deploy:deploy'
            }
        }
*/		
		stage('OWASP - Dependencies check') {
            steps {
                echo 'Check OWASP dependencies ...'
				sh 'mvn dependency-check:check'
            }
        }
		
		stage('Sonar - Code Quality') {
            steps {
                echo 'Check Code Quality ...'
				sh 'mvn sonar:sonar' // -Dsonar.dependencyCheck.reportPath=target/dependency-check-report.xml'
            }
        }

/*		
        stage('Contract testing') {
            steps {
                echo 'Testing application conformity according to its Swagger definition ...'
            }
        }
*/
        stage('Bake') {
            steps {
			    sh 'echo \"Verification de la presence de l\'image Docker dans la registry locale (elle a du avoir le temps de se reconstruire ou se telecharger)\"'
				sh 'timeout 60 sh -c \'until docker images | grep $dockerRepo | grep $DOCKER_CACHE_IMAGE_VERSION; do sleep 1; done\''
				sh 'echo \"Modification du dockerfile pour y indiquer l\'image de base a utiliser pour le build afin de beneficier des layer mis en cache localement\"'
				sh 'sed -i.bak \"s#BASIS_IMAGE#$dockerRegistry/$dockerRepo:$DOCKER_CACHE_IMAGE_VERSION#g\" dockerfile'
                echo 'Building Docker image ...'
				sh '$(aws ecr get-login --no-include-email --region $AWS_REGION)'
				sh 'docker build --build-arg PACKAGE_VERSION=${package_version} --build-arg APPLICATION_NAME=${applicationName} -t ${dockerRegistry}/${dockerRepo}:${package_version} .'
				//echo 'Removing dangling Docker image from the local registry ...'
				//sh "docker rmi $(docker images --filter "dangling=true" -q --no-trunc) 2>/dev/null"
				echo 'Publishing Docker image into the private registry ...'
				sh 'docker push ${dockerRegistry}/${dockerRepo}:${package_version}'
            }
        }
    }

}