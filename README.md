# Pipeline Script
```groovy
pipeline {
    agent any

    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_USERNAME = 'asa96'
        APP_NAME = "netflix-clone-app"
        IMAGE_NAME = "${DOCKER_USERNAME}/${APP_NAME}"
    }
    
    stages {

        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
		
        stage('Git Checkout'){
            steps{
                git branch: 'main', url: 'https://github.com/saeedalig/Netflix-Clone-App.git'
            }
        }
		
	stage("Sonarqube Analysis"){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
		
        stage("Quality Gate Status"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
		
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
		
        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
		
        stage('TRIVY FS Scan') {
            steps {
                sh "trivy fs . > trivy-fs.txt"
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build docker image using TMDB_V3_API_KEY stored in credentials
                    withCredentials([string(credentialsId: 'netflix-api-key', variable: 'TMDB_V3_API_KEY')]) {
                        sh "docker build --build-arg TMDB_V3_API_KEY=$TMDB_V3_API_KEY -t $IMAGE_NAME ."
                        sh "docker tag $IMAGE_NAME $IMAGE_NAME:${BUILD_ID}"
                        sh "docker tag $IMAGE_NAME $IMAGE_NAME:latest"
                    }
                }
            }
        }
		
	stage("TRIVY Image Scan"){
            steps{
                sh "trivy image asa96/netflix-clone-app:latest > trivy-image.txt"
            }
        }
		
        stage('Docker Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-auth', passwordVariable: 'PASSWD', usernameVariable: 'USER')]) {
                        sh "docker login -u ${env.USER} -p ${env.PASSWD}"
                        sh "docker push $IMAGE_NAME:${BUILD_ID}"
                        sh "docker push $IMAGE_NAME:latest"
                    }
                }
            }
        }
		
	stage('Delete Docker Images'){
            steps {
                sh "docker rmi $IMAGE_NAME:${BUILD_ID}"
                sh "docker rmi ${IMAGE_NAME}:latest"
            }
        }
		
		stage('Update k8s deployment file') {
			steps {
				script {
					// Navigate to manifest directory
					dir('kubernetes') {
						// Display the content of the deployment.yml before modification
						sh "cat deployment.yml"

						// Update the image tag in deployment.yml
						sh "sed -i \"s|\\(image:.*${APP_NAME}\\).*|\\1:${BUILD_ID}|\" deployment.yml"

						// Display the content of the deployment.yml after modification
						sh "cat deployment.yml"
					}
				}
			}
		}
		
		stage('Push the changed deployment file to GitHub') {
			steps {
				script {
					// Configure Git user information
					sh """
						git config --global user.name "zeal"
						git config --global user.email "zeal@gmail.com"
					"""

					// Add the changed deployment file to the Git repository
					sh 'git add kubernetes/deployment.yml'

					// Commit the changes
					sh 'git commit -m "Updated the deployment file"'

					// Push the changes to the GitHub repository
					withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
						sh 'git remote set-url origin https://$USER:$PASS@github.com/saeedalig/youtube-clone-app.git'
						sh 'git push origin main'
					}
				}
			}
		}
		post {
		always {
			emailext attachLog: true,
				subject: "'${currentBuild.result}'",
				body: "Project: ${env.JOB_NAME}<br/>" +
					"Build Number: ${env.BUILD_NUMBER}<br/>" +
					"URL: ${env.BUILD_URL}<br/>",
				to: 'zeal999@gmail.com',
				attachmentsPattern: 'trivy-fs.txt,trivy-image.txt'
			}
		}
	}
}
```
