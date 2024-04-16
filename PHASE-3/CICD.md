## Install Plugins in Jenkins

1. **Eclipse Temurin Installer**:
   - This plugin enables Jenkins to automatically install and configure the Eclipse Temurin JDK (formerly known as AdoptOpenJDK).
   - To install, go to Jenkins dashboard -> Manage Jenkins -> Manage Plugins -> Available tab.
   - Search for "Eclipse Temurin Installer" and select it.
   - Click on the "Install without restart" button.

2. **Pipeline Maven Integration**:
   - This plugin provides Maven support for Jenkins Pipeline.
   - It allows you to use Maven commands directly within your Jenkins Pipeline scripts.
   - To install, follow the same steps as above, but search for "Pipeline Maven Integration" instead.

3. **Config File Provider**:
   - This plugin allows you to define configuration files (e.g., properties, XML, JSON) centrally in Jenkins.
   - These configurations can then be referenced and used by your Jenkins jobs.
   - Install it using the same procedure as mentioned earlier.

4. **SonarQube Scanner**:
   - SonarQube is a code quality and security analysis tool.
   - This plugin integrates Jenkins with SonarQube by providing a scanner that analyzes code during builds.
   - You can install it from the Jenkins plugin manager as described above.

5. **Kubernetes CLI**:
   - This plugin allows Jenkins to interact with Kubernetes clusters using the Kubernetes command-line tool (`kubectl`).
   - It's useful for tasks like deploying applications to Kubernetes from Jenkins jobs.
   - Install it through the plugin manager.

6. **Kubernetes**:
   - This plugin integrates Jenkins with Kubernetes by allowing Jenkins agents to run as pods within a Kubernetes cluster.
   - It provides dynamic scaling and resource optimization capabilities for Jenkins builds.
   - Install it from the Jenkins plugin manager.

7. **Docker**:
   - This plugin allows Jenkins to interact with Docker, enabling Docker builds and integration with Docker registries.
   - You can use it to build Docker images, run Docker containers, and push/pull images from Docker registries.
   - Install it from the plugin manager.

8. **Docker Pipeline Step**:
   - This plugin extends Jenkins Pipeline with steps to build, publish, and run Docker containers as part of your Pipeline scripts.
   - It provides a convenient way to manage Docker containers directly from Jenkins Pipelines.
   - Install it through the plugin manager like the others.

After installing these plugins, you may need to configure them according to your specific environment and requirements. This typically involves setting up credentials, configuring paths, and specifying options in Jenkins global configuration or individual job configurations. Each plugin usually comes with its own set of documentation to guide you through the configuration process.

## Configure Above Plugins in Jenkins as per video

## Pipeline 

```groovy

pipeline {
    agent any
    tools {
        jdk 'jdk-17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
    }
    stages {
        stage('Check Out Code') {
            steps {
                git branch: 'main', credentialsId: 'gitCred', url: 'https://github.com/Bicky7735369355/java-web-app.git'
            }
        }
        stage ('compile') {
            steps {
                sh 'mvn compile'
            }
        }
        stage ('Test') {
            steps {
                sh 'mvn test'
            }
            
        }
        stage ('file system scan') {
            steps {
                sh 'trivy fs --format table -o fs-report.html .'
            }
        }
        stage ('Code analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn sonar:sonar'
                }
                
            }
                
        }
        stage ('Quality gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false , credentialsId: 'sonarCred'
                }
            }
        }
        stage ('Build Code') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage ('Publish Artifact') {
            steps {
                withMaven(globalMavenSettingsConfig: 'my-global-setting', jdk: 'jdk-17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                
                    sh 'mvn deploy'
                }
            }
        }
        stage ('Build and tag docker image') {
            steps {
                sh 'docker build -t bicky77/finally .'
            }
        }
        stage ('Scan Docker Image' ) {
            steps {
                sh 'trivy image --format table -o image-report.html bicky77/finally'
            }
        }
        stage ('Push docker image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerCred', toolName: 'docker') {
                        sh 'docker push bicky77/finally'
                    }
                }
                    
            }
        }
        stage ('Deploy to container') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8sCred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.38.55:6443') {
                    sh 'kubectl apply -f k8s-deploy.yml'
                    
                }
            }
        }
        stage ('Verify to container') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8sCred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.38.55:6443') {
                    sh 'kubectl get pods'
                    sh 'kubectl get svc'
                    
                }
            }
        }
    }        
        
    }
    post {
    always {
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'jaiswaladi246@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}

}
```
