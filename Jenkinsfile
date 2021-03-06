//Picking a build agent labeled "ec2" to run pipeline on
node ('ec2'){
  stage 'Pull from SCM'  
	//Passing the pipeline the ID of my GitHub credentials and specifying the repo for my app
	git credentialsId: 'GitHub-Amit', url: 'https://github.com/amitbitcse/game-of-life.git'
	stage 'Test Code'  
	sh 'mvn install'

  stage 'Build app' 
	//Running the maven build and archiving the war
	sh 'mvn install -Dmaven.test.skip=true'
	archive 'target/*.war'
  
  stage 'Package Image'
	//Packaging the image into a Docker image
	def pkg = docker.build ('game-of-life', '.')

  
  stage 'Push Image to DockerHub'
	//Pushing the packaged app in image into DockerHub
	//docker.withRegistry ('https://index.docker.io/v1/', 'DockerRegistry-Amit') {
	docker.withRegistry ('https://686703771370.dkr.ecr.us-east-1.amazonaws.com', 'ecr:AWS-Amit') {
	  sh 'ls -lart' 
	  pkg.push 'v_${BUILD_NUMBER}'
	}
  
  stage 'Stage image'
	//Deploy image to staging in ECS
  
	// Create Json file with build number from template json file
	sh 'sed -e "s;%BUILD_NUMBER%;${BUILD_NUMBER};g" GameOfLife-Task.json > GameOfLife-Task-v_${BUILD_NUMBER}.json'

	// Create New Task Defition using above created json file with latest
	sh "aws ecs register-task-definition --family GameOfLife-Task --cli-input-json file://GameOfLife-Task-v_${env.BUILD_NUMBER}.json"

	// Update Service with new Task Definition
	sh 'aws ecs describe-task-definition --task-definition GameOfLife-Task | egrep "revision" | cut -d":" -f2 | sed "s/ //g" > .GameOfLife-Task-Revision'
	def TASK_REVISION = readFile(".GameOfLife-Task-Revision")
	println "${TASK_REVISION}"

	sh "aws ecs update-service --service Staging-GameOfLife-Service  --cluster Staging-GameOfLife-Cluster --desired-count 0"
	timeout(time: 5, unit: 'MINUTES') {
		waitUntil {
			sh "aws ecs describe-services --service Staging-GameOfLife-Service  --cluster Staging-GameOfLife-Cluster  > .amazon-ecs-service-status.json"

			// parse `describe-services` output
			def ecsServicesStatusAsJson = readFile(".amazon-ecs-service-status.json")
			def ecsServicesStatus = new groovy.json.JsonSlurper().parseText(ecsServicesStatusAsJson)
			println "$ecsServicesStatus"
			def ecsServiceStatus = ecsServicesStatus.services[0]
			return ecsServiceStatus.get('runningCount') == 0 && ecsServiceStatus.get('status') == "ACTIVE"
		}
	}
	sh "aws ecs update-service --service Staging-GameOfLife-Service  --cluster Staging-GameOfLife-Cluster  --desired-count 1  --task-definition GameOfLife-Task:${TASK_REVISION}"
	timeout(time: 5, unit: 'MINUTES') {
		waitUntil {
			sh "aws ecs describe-services --service Staging-GameOfLife-Service --cluster Staging-GameOfLife-Cluster > .amazon-ecs-service-status.json"

			// parse `describe-services` output
			def ecsServicesStatusAsJson = readFile(".amazon-ecs-service-status.json")
			def ecsServicesStatus = new groovy.json.JsonSlurper().parseText(ecsServicesStatusAsJson)
			println "$ecsServicesStatus"
			def ecsServiceStatus = ecsServicesStatus.services[0]
			return ecsServiceStatus.get('runningCount') == 1 && ecsServiceStatus.get('status') == "ACTIVE"
		}
	}
	timeout(time: 5, unit: 'MINUTES') {
		waitUntil {
			try {
				sh "curl http://staging-gameoflife-elb-1356454014.us-east-1.elb.amazonaws.com/"
				return true
			} catch (Exception e) {
				return false
			}
		}
	}
	echo "gameoflife#${env.BUILD_NUMBER} SUCCESSFULLY deployed to http://staging-gameoflife-elb-1356454014.us-east-1.elb.amazonaws.com/"
	input 'Does staging http://staging-gameoflife-elb-1356454014.us-east-1.elb.amazonaws.com/ look okay?'

  stage 'Deploy to ECS'
	//Deploy image to production in ECS
	sh "aws ecs update-service --service Production-GameOfLife-Service  --cluster Production-GameOfLife-Cluster --desired-count 0"
	timeout(time: 5, unit: 'MINUTES') {
		waitUntil {
			sh "aws ecs describe-services --service Production-GameOfLife-Service  --cluster Production-GameOfLife-Cluster   > .amazon-ecs-service-status.json"

			// parse `describe-services` output
			def ecsServicesStatusAsJson = readFile(".amazon-ecs-service-status.json")
			def ecsServicesStatus = new groovy.json.JsonSlurper().parseText(ecsServicesStatusAsJson)
			println "$ecsServicesStatus"
			def ecsServiceStatus = ecsServicesStatus.services[0]
			return ecsServiceStatus.get('runningCount') == 0 && ecsServiceStatus.get('status') == "ACTIVE"
		}
	}
	sh "aws ecs update-service --service Production-GameOfLife-Service  --cluster Production-GameOfLife-Cluster  --desired-count 1  --task-definition GameOfLife-Task:${TASK_REVISION}"
	timeout(time: 5, unit: 'MINUTES') {
		waitUntil {
			sh "aws ecs describe-services --service Production-GameOfLife-Service  --cluster Production-GameOfLife-Cluster  > .amazon-ecs-service-status.json"

			// parse `describe-services` output
			def ecsServicesStatusAsJson = readFile(".amazon-ecs-service-status.json")
			def ecsServicesStatus = new groovy.json.JsonSlurper().parseText(ecsServicesStatusAsJson)
			println "$ecsServicesStatus"
			def ecsServiceStatus = ecsServicesStatus.services[0]
			return ecsServiceStatus.get('runningCount') == 1 && ecsServiceStatus.get('status') == "ACTIVE"
		}
	}
	timeout(time: 5, unit: 'MINUTES') {
		waitUntil {
			try {
				sh "curl http://production-gameoflife-elb-1933452439.us-east-1.elb.amazonaws.com/"
				return true
			} catch (Exception e) {
				return false
			}
		}
	}
	echo "gameoflife#${env.BUILD_NUMBER} SUCCESSFULLY deployed to http://production-gameoflife-elb-1933452439.us-east-1.elb.amazonaws.com/"
}
