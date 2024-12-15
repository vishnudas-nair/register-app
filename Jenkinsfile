pipeline {
    agent any
        
    parameters{
        string(name: 'DOCKER_TAG', defaultValue: 'latest', description: "Docker tag")
    }
    
    tools {
        maven 'maven3'
    }

    // environment {
    //     SCANNER_HOME = tool 'sonar-scanner'
    // }

    stages {
        stage('Git checkout') {
            steps {
                git branch: 'main', credentialsId: 'github-vishnu', url: 'https://github.com/vishnudas-nair/register-app.git'
            }
        }
        
         stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }
        
        
        
         stage('Unit test cases') {
            steps {
                sh 'mvn test -DskipTests=true'
            }
        }
        
        
         stage('Scan Dependencies') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }
        
        stage('Build') {
            steps {
                script {
                    // Maven build using Jenkins Maven plugin
                    sh 'mvn clean package'
                }
            }
        }
        
        stage("SonarQube Analysis"){
          steps {
	           script {
		        withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') { 
                        sh "mvn sonar:sonar"
		        }
	           }	
          }
      }
      
      stage("Quality Gate"){
           steps {
               script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }	
            }

        }
        
        stage('Docker Build and Tag') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker') {
                        sh '''
                        docker build -t vishnudasnair619/argocd-maven:${DOCKER_TAG} .
                        '''
                    }
                }
            }
        }
        
        stage('Docker Image scan') {
            steps {
                sh 'trivy image --format table -o dimage.html vishnudasnair619/argocd-maven:${DOCKER_TAG}'
            }
        }
        
        stage('Docker Push') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker push vishnudasnair619/argocd-maven:${DOCKER_TAG}"
                    }
                }
            }
        }
        
        stage('Update YAML Manifests in GitOps Repo') {
    steps {
        script {
            withCredentials([gitUsernamePassword(credentialsId: 'github-vishnu', gitToolName: 'Default')]) {
                sh '''
                set -e  # Exit on error

                # Clean up any existing directory and clone the repository
                echo "Cloning the GitOps repository..."
                rm -rf gitops-register-app
                git clone https://github.com/vishnudas-nair/gitops-register-app.git || { echo "Failed to clone repo"; exit 1; }

                # Navigate to the repo directory
                cd gitops-register-app || { echo "Directory not found"; exit 1; }

                # Validate if the YAML files exist
                if [ ! -f deployment.yaml ]; then
                    echo "YAML file not found: deployment.yaml"
                    exit 1
                fi

                if [ ! -f service.yaml ]; then
                    echo "YAML file not found: service.yaml"
                    exit 1
                fi

                # Update the deployment YAML file with the new Docker tag
                sed -i 's|image: vishnudasnair619/argocd-maven:.*|image: vishnudasnair619/argocd-maven:'${DOCKER_TAG}'|' deployment.yaml || { echo "Failed to update deployment YAML"; exit 1; }

                # Verify the update
                echo "Updated deployment.yaml file contents:"
                cat deployment.yaml

                # Optional: Make changes to service.yaml if needed
                # Example: sed command for updating fields in service.yaml

                # Configure Git credentials
                git config user.email "vishnudasnair619@gmail.com"
                git config user.name "vishnudas-nair"

                # Commit and push the changes
                git add deployment.yaml service.yaml
                git commit -m "Update manifests with Docker tag ${DOCKER_TAG}" || echo "No changes to commit"
                git push origin main || { echo "Failed to push changes"; exit 1; }
                '''
            }
        }
    }
}             
    }
}
