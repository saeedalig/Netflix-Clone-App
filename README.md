# Deploying Netflix Clone App using CICD pipeline.
The Netflix Clone project aims to replicate the core functionalities of the popular streaming service, Netflix.

<div align="center">
  <img src="./public/assets/home-page.png" alt="Logo" width="100%" height="100%">
  <p align="center">Home Page</p>
</div>

- Implementing a Continuous Integration and Continuous Deployment (CI/CD) pipeline using Jenkins. This pipeline automates the building, testing, and deployment processes, ensuring efficient and reliable software delivery.

- Integrated GitOps principles with ArgoCD for managing and automating Kubernetes deployments through Git repositories. This approach enhances collaboration and ensures consistency in the deployment process.

## Branching Strategy:
- Choosing an appropriate branching strategy for CI/CD is essential for a smooth development and release process. The choice of branching strategy often depends on factors such as team size, project complexity, release frequency, and collaboration needs.

- For simplicity, I have created two branches `main` and `staging`.Users will push all changes to the `staging branch`. The owner of the `main branch` wiil first review the chnages and then merge the chnages into `main branch` by raising a ***Pull Request***.

- As soon as the PR is opened, the `webhook` integrated with Jenkins, triggers the Jenkins Job and perform the specified tasks.

<div align="center">
  <img src="https://miro.medium.com/v2/resize:fit:828/format:webp/0*S2QvGHThfniSkkQA" alt="Logo" width="100%" height="100%">
  <p align="center">Home Page</p>
</div>

- ***Version Control (GitHub):*** Used GitHub for source code repository, enabling version control and collaboration among development teams.

- ***Webhook:*** Implemented webhooks in GitHub to trigger events and notify Jenkins about changes in the repository, automating the CI/CD pipeline.

- ***Jenkins:*** Jenkins is employed for continuous integration, automating the build and test processes whenever changes are pushed to the GitHub repository.

- ***Code Quality (SonarQube):*** Integrated SonarQube to analyze code quality, identify bugs, and enforce coding standards as part of the CI/CD process.

- ***Security Scanning (OWASP, Trivy):*** Incorporated OWASP Dependency-Check and Trivy for security scanning, identifying and addressing vulnerabilities in dependencies and container images.

- ***Containerization (Docker):*** Utilized Docker for containerization, encapsulating the application and its dependencies for consistent deployment across environments.

- ***Orchestration (Kubernetes):*** Employed Kubernetes for container orchestration, providing scalability, resilience, and automated management of containerized applications.

- ***GitOps (ArgoCD):*** Implemented ArgoCD for GitOps-based continuous delivery, managing Kubernetes resources declaratively through Git repositories.

- ***Monitoring (Prometheus and Grafana):*** Integrated Prometheus for collecting and storing metrics, and Grafana for visualizing and analyzing these metrics to monitor the health and performance of the application.

- ***Email Notification:*** Configured email notifications in Jenkins to notify stakeholders about the status of builds and deployments, facilitating communication within the development team.

You can go through the server setup for pipeline by following the link https://github.com/saeedalig/DevOps-Project/tree/main/Server-Setup-For-Pipeline.
Feel free to use by customizing it as per your needs.

# Pipeline Script

```
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
				attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
			}
		}
	}
}
```
