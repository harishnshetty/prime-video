pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Git Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/harishnshetty/prime-video.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=amazon-prime \
                        -Dsonar.projectKey=amazon-prime '''
                }
            }
        }

        stage("Quality Gate") {
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

        // stage("OWASP FS Scan") {
        //     steps {
        //         dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
        //                         odcInstallation: 'dp-check'
        //         dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        //     }
        // }


        stage("OWASP FS Scan") {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan ./ 
                    --disableYarnAudit 
                    --disableNodeAudit 
                    --nvdApiKey 66461f9e-56f3-49eb-89b5-46a61b58dc09
                    ''',
                odcInstallation: 'dp-check'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }


        stage("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Build Docker Image") {
            steps {
                script {
                    env.IMAGE_TAG = "harishnshetty/amazon-prime:${BUILD_NUMBER}"

                    // Optional cleanup
                    sh "docker rmi -f amazon-prime ${env.IMAGE_TAG} || true"

                    sh "docker build -t amazon-prime ."
                }
            }
        }

        stage("Tag & Push to DockerHub") {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker-cred', variable: 'dockerpwd')]) {
                        sh "docker login -u harishnshetty -p ${dockerpwd}"
                        sh "docker tag amazon-prime ${env.IMAGE_TAG}"
                        sh "docker push ${env.IMAGE_TAG}"

                        // Also push latest
                        sh "docker tag amazon-prime harishnshetty/amazon-prime:latest"
                        sh "docker push harishnshetty/amazon-prime:latest"
                    }
                }
            }
        }

       

        stage("Trivy Scan Image") {
            steps {
                script {
                    sh """
                    echo '🔍 Running Trivy scan on ${env.IMAGE_TAG}'

                    # JSON report
                    trivy image -f json -o trivy-report.json ${env.IMAGE_TAG}

                    # HTML report using built-in HTML format
                    trivy image -f table -o trivy-report.txt ${env.IMAGE_TAG}

                    # Fail build if HIGH/CRITICAL vulnerabilities found
                    # trivy image --exit-code 1 --severity HIGH,CRITICAL ${env.IMAGE_TAG} || true
                """
                }
            }
        }


        stage("Deploy to Container") {
            steps {
                script {
                    sh "docker rm -f amazon-prime || true"
                    sh "docker run -d --name amazon-prime -p 3000:3000 ${env.IMAGE_TAG}"
                }
            }
        }
    }

     post {
		always {
		    emailext(
		        to: 'harishn662@gmail.com',
		        subject: "📢 Jenkins Build Report: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
		        body: """
		            <html>
		                <body style="font-family: Arial, sans-serif; line-height: 1.5;">
		                    <p>📌 <b>This is a Jenkins BINGO CICD pipeline status.</b></p>
		                    <p><b>Project:</b> ${env.JOB_NAME}</p>
		                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
		                    <p><b>Build Status:</b> ${env.buildStatus}</p>
		                    <p><b>Started by:</b> ${env.buildUser}</p>
		                    <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
		                </body>
		            </html>
		        """,
		        mimeType: 'text/html',
		        attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
		    )
		}
    }

}


