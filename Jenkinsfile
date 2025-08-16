pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        NVD_API_KEY = credentials('NVD_API_KEY')
    }
    stages {
        stage ("clean workspace") {
            steps {
                cleanWs()
            }
        }
        stage ("Git checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/harishnshetty/prime-video.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=amazon-prime \
                    -Dsonar.projectKey=amazon-prime '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            } 
        }
        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        } 
        
   //     stage('OWASP FS SCAN') {
   // steps {
    //    withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
     //       dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey $NVD_API_KEY", odcInstallation: 'dp-check'
      //      dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
       // }
  //  }
// }

        stage ("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivy.txt"
            }
        }
        stage("Build Docker Image") {
    steps {
        script {
            // Define unique tag using BUILD_NUMBER
            env.IMAGE_TAG = "harishnshetty/amazon-prime:${BUILD_NUMBER}"

            // Optional: remove old local image if exists
            sh "docker rmi -f amazon-prime ${env.IMAGE_TAG} || true"

            // Build Docker image
            sh "docker build -t amazon-prime ."
        }
    }
}

stage("Tag & Push to DockerHub") {
    steps {
        script {
            withDockerRegistry(credentialsId: 'docker-cred') {
                // Tag with unique IMAGE_TAG
                sh "docker tag amazon-prime ${env.IMAGE_TAG}"

                // Push unique tag
                sh "docker push ${env.IMAGE_TAG}"

                // Optional: also push latest if needed
                sh "docker tag amazon-prime harishnshetty/amazon-prime:latest"
                sh "docker push harishnshetty/amazon-prime:latest"
            }
        }
    }
}

//         stage('Docker Scout Image') {
//     steps {
//         script {
//             sh """
//                 mkdir -p ${WORKSPACE}/docker-scout-cache

//                 DOCKER_SCOUT_CACHE=${WORKSPACE}/docker-scout-cache docker-scout quickview ${env.IMAGE_TAG}
//                 DOCKER_SCOUT_CACHE=${WORKSPACE}/docker-scout-cache docker-scout cves ${env.IMAGE_TAG}
//                 DOCKER_SCOUT_CACHE=${WORKSPACE}/docker-scout-cache docker-scout recommendations ${env.IMAGE_TAG}
//             """
//         }
//     }
// }



        stage("Deploy to Container") {
    steps {
        script {
            // Remove old container if exists
            sh "docker rm -f amazon-prime || true"

            // Run new container
            sh "docker run -d --name amazon-prime -p 3000:3000 ${env.IMAGE_TAG}"
        }
    }
}

    }
    post {
    always {
        script {
            emailext(
                attachLog: true,
                subject: "Build Result: ${currentBuild.result}",
                body: """
                    <html>
                    <body>
                        <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                            <p style="color: white; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                        </div>
                        <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                            <p style="color: white; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                        </div>
                        <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px;">
                            <p style="color: white; font-weight: bold;">URL: <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>
                        </div>
                    </body>
                    </html>
                """,
                to: 'harishn662@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy.txt'
            )
        }
    }
}

}
