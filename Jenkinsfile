/*
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout From Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Aj7Ay/Petclinic-Real.git'
            }
        }
        stage('mvn compile'){
            steps{
                sh 'mvn clean compile'
            }
        }
        stage('mvn test'){
            steps{
                sh 'mvn test'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Petclinic \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=Petclinic '''
                }
            }
        }
        stage("quality gate"){
           steps {
                 script {
                     waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                    }
                } 
        } 
        stage('mvn build'){
            steps{
                sh 'mvn clean install'
            }
        }  
        stage("OWASP Dependency Check"){
            steps{
                dependencyCheck additionalArguments: '--scan ./ --format HTML ', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: 'dependency-check-report.html'
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build -t petclinic1 ."
                       sh "docker tag petclinic1 sevenajay/petclinic1:latest "
                       sh "docker push sevenajay/petclinic1:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image sevenajay/petclinic1:latest > trivy.txt" 
            }
        }
        stage('Clean up containers') {   //if container runs it will stop and remove this block
          steps {
           script {
             try {
                sh 'docker stop pet1'
                sh 'docker rm pet1'
                } catch (Exception e) {
                  echo "Container pet1 not found, moving to next stage"  
                }
            }
          }
        }
        stage ('Manual Approval'){
          steps {
           script {
             timeout(time: 10, unit: 'MINUTES') {
              def approvalMailContent = """
              Project: ${env.JOB_NAME}
              Build Number: ${env.BUILD_NUMBER}
              Go to build URL and approve the deployment request.
              URL de build: ${env.BUILD_URL}
              """
             mail(
             to: 'postbox.aj99@gmail.com',
             subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", 
             body: approvalMailContent,
             mimeType: 'text/plain'
             )
            input(
            id: "DeployGate",
            message: "Deploy ${params.project_name}?",
            submitter: "approver",
            parameters: [choice(name: 'action', choices: ['Deploy'], description: 'Approve deployment')]
            )  
          }
         }
       }
    }
        stage('Deploy to conatiner'){
            steps{
                sh 'docker run -d --name pet1 -p 8082:8080 sevenajay/petclinic1:latest'
            }
        }
        stage("Deploy To Tomcat"){
            steps{
                sh "sudo cp  /var/lib/jenkins/workspace/petclinic/target/petclinic.war /opt/apache-tomcat-9.0.65/webapps/ "
            }
        }
        stage('Deploy to kubernets'){
            steps{
                script{
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                       sh 'kubectl apply -f deployment.yaml'
                  }
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
            to: 'postbox.aj99@gmail.com',
            attachmentsPattern: 'trivy.txt'
        }
    }
}


// try this approval stage also 

stage('Manual Approval') {
  timeout(time: 10, unit: 'MINUTES') {
    mail to: 'postbox.aj99@gmail.com',
         subject: "${currentBuild.result} CI: ${env.JOB_NAME}",
         body: "Project: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nGo to ${env.BUILD_URL} and approve deployment"
    input message: "Deploy ${params.project_name}?", 
           id: "DeployGate", 
           submitter: "approver", 
           parameters: [choice(name: 'action', choices: ['Deploy'], description: 'Approve deployment')]
  }
}
*/

pipeline{
    agent any
    
    tools{
        maven 'maven3.9.11'
    }
    
    triggers {
        githubPush()
    }
    
    options {
        buildDiscarder(logRotator(
            artifactDaysToKeepStr: '4',
            artifactNumToKeepStr: '3',
            daysToKeepStr: '5',
            numToKeepStr: '4'
        ))
        timestamps()
    }
    
    stages{
        stage('CheckoutCode'){
            steps{
                git credentialsId: '81121b4f-64b3-46db-8908-dc33cb0edb99', url: 'https://github.com/akhilmatta89/Petclinic.git'
            }
        }
        stage('BuildPackage'){
            steps{
                sh "mvn clean package"
            }
        }
        
        stage ('Sonar'){
            steps{
                sh "mvn sonar:sonar"
            }
        }
        
        stage ('UploadArtifactsToNexus'){
            steps{
                sh "mvn deploy"
            }
        }
        
        stage ('DeployToTomCat'){
            steps{
               sshagent(['6b398c92-3ef8-4e48-868c-d1626f8cac61']) {
                sh "scp -o StrictHostKeyChecking=no target/petclinic.war ec2-user@13.61.186.132:/opt/apache-tomcat-9.0.109/webapps/"
                }  
            }
        }
    }
post {
        always {
            emailext(
                subject: "Build #${BUILD_NUMBER} - ${JOB_NAME} - ${currentBuild.currentResult}",
                body: """
                <html>
                <body>
                    <h2>Jenkins Build Notification</h2>
                    <p><b>Job:</b> ${JOB_NAME}</p>
                    <p><b>Build Number:</b> ${BUILD_NUMBER}</p>
                    <p><b>Status:</b> ${currentBuild.currentResult}</p>
                    <p><b>Build URL:</b> <a href="${BUILD_URL}">${BUILD_URL}</a></p>
                    <br/>
                    <p>Regards,<br/>Akhil Reddy</p>
                </body>
                </html>
                """,
                from: "jenkins@akhil.com",
                to: "akhilreddy2672@gmail.com,likhitha.telukuntla3@gmail.com",
                mimeType: 'text/html'
            )
        }
    }
}
