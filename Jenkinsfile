pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
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
        
       stage('OWASP FS SCAN') {
   steps {
       dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'dp-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
       }
   }
}

        stage ("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
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

stage('Trivy Scan Image') {
    steps {
        script {
            sh """
                echo '🔍 Running Trivy scan on ${env.IMAGE_TAG}'

                # Generate reports
                trivy image -f json -o trivy-report.json ${env.IMAGE_TAG}
                trivy image -f template --template "@contrib/html.tpl" -o trivy-report.html ${env.IMAGE_TAG}

                # Fail build if HIGH/CRITICAL found
                trivy image --exit-code 1 --severity HIGH,CRITICAL ${env.IMAGE_TAG} || true
            """
        }
    }
}


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
        archiveArtifacts artifacts: 'trivy-report.*', fingerprint: true
        emailext(
            to: 'harishn662@gmail.com',
            subject: "📢 Jenkins Build Report: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: """
                <html>
                    <body style="font-family: Arial, sans-serif; line-height: 1.5;">
                        <p>📌 <b>This is a Jenkins BINGO CICD pipeline status.</b></p>
                        <p><b>Project:</b> ${env.JOB_NAME}</p>
                        <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                        <p><b>Build Status:</b> ${currentBuild.currentResult}</p>
                        <p><b>Started by:</b> ${BUILD_USER ?: "N/A"}</p>
                        <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    </body>
                </html>
            """,
            mimeType: 'text/html',
            attachmentsPattern: 'trivy-report.*,trivyfs.txt, dependency-check-report.xml'
        )
    }
}

