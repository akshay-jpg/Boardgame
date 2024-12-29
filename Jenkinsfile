pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool(name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation')
    }
    
    parameters {
        string(name: 'DOCKER_IMAGE_TAG', defaultValue: '', description: 'Setting docker image for latest push')
    }
    
    stages {
        stage("Validate Parameters") {
            steps {
                script {
                    if (params.DOCKER_IMAGE_TAG == '') {
                        error("DOCKER_IMAGE_TAG must be provided.")
                    }
                }
            }
        }
        
        stage("Workspace cleanup"){
            steps{
                script{
                    cleanWs()
                }
            }
        }
        
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/akshay-jpg/Boardgame.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' 
                            $SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=mega-project-akshay \
                            -Dsonar.projectKey=mega-project-akshay \
                            -Dsonar.java.binaries=. 
                        '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        
        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true)
                    {sh "mvn deploy"}
            }
        }
        
        stage('Build and tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker')
                          {  sh "docker build -t akshaybajait/boardshack:${params.DOCKER_IMAGE_TAG} ."}
                }
            }
        }
        
        stage('Dcoker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html akshaybajait/boardshack:${params.DOCKER_IMAGE_TAG} "
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker')
                          {  sh "docker push akshaybajait/boardshack:${params.DOCKER_IMAGE_TAG}" }
                }
            }
        }
        
        stage("Update: Kubernetes manifests"){
            steps{
                script{
                        sh """
                            sed -i -e s/boardshack.*/boardshack:${params.DOCKER_IMAGE_TAG}/g deployment-service.yaml
                        """
                    
                }
            }
        }
        
        stage("Git: Code update and push to GitHub"){
            steps{
                script{
                    withCredentials([gitUsernamePassword(credentialsId: 'git-cred', gitToolName: 'Default')]) {
                        sh '''
                        echo "Checking repository status: "
                        git status
                    
                        echo "Adding changes to git: "
                        git add .
                        
                        echo "Commiting changes: "
                        git commit -m "Kubernetes manifest file update successfully..."
                        
                        echo "Pushing changes to github: "
                        git push https://github.com/akshay-jpg/Boardgame.git main
                    '''
                    }
                }
            }
        }
         

     
        }
        
    post {
        success {
            script {
                
                emailext attachLog: true,
                from: 'akshaybajait@gmail.com',
                subject: "Akshay-Mega-Project Application has been updated and deployed - '${currentBuild.result}'",
                body: """
                    <html>
                    <body>
                        <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                            <p style="color: black; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                        </div>
                        <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                            <p style="color: black; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                        </div>
                        <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px;">
                            <p style="color: black; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                        </div>
                    </body>
                    </html>
            """,
            to: 'akshaybajait@gmail.com',
            mimeType: 'text/html'
            }
        }
        failure {
            script {
                
                emailext attachLog: true,
                from: 'akshaybajait@gmail.com',
                subject: "Akshay-Mega-Project Application build failed - '${currentBuild.result}'",
                body: """
                    <html>
                    <body>
                        <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                            <p style="color: black; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                        </div>
                        <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                            <p style="color: black; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                        </div>
                    </body>
                    </html>
            """,
            to: 'akshaybajait@gmail.com',
            mimeType: 'text/html'
            }
        }
    }
}
