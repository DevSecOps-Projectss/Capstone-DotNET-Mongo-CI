pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_REPO = 'hamudevops/noteapp'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-token',
                    url: 'https://github.com/DevSecOps-Projectss/Capstone-DotNET-Mongo-CI.git'
            }
        }

        stage('Gitleaks Scan') {
            steps {
                sh 'gitleaks detect --report-format=json --report-path=gitleaks-report.json --exit-code=1'
            }
        }

        stage('Compile') {
            steps {
                sh 'dotnet build'
            }
        }

        stage('trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('Unit Testing') {
            steps {
                sh 'dotnet test --logger "trx;LogFileName=test-results.trx"'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=NoteApp \
                        -Dsonar.projectKey=NoteApp'''
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Build Image & Tag Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ${IMAGE_REPO}:${IMAGE_TAG} ."
                    }
                }
            }
        }

        stage('trivy Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html ${IMAGE_REPO}:${IMAGE_TAG}"
            }
        }

        stage('Push Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${IMAGE_REPO}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Update Manifest File CD Repo') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(credentialsId: 'git-token', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')
                    ]) {
                        sh '''
                            set -e
                            rm -rf Capstone-DotNET-Mongo-CD
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/DevSecOps-Projectss/Capstone-DotNET-Mongo-CD.git
                            cd Capstone-DotNET-Mongo-CD

                            # update the image tag in the manifest
                            sed -i "s|image: .*noteapp:.*|image: hamudevops/noteapp:${IMAGE_TAG}|" Manifest/manifest.yaml

                            echo "Updated manifest file contents:"
                            cat Manifest/manifest.yaml

                            git config user.name "Jenkins"
                            git config user.email "jenkins@yourdomain.com"
                            git add Manifest/manifest.yaml
                            git commit -m "Update image tag to ${IMAGE_TAG}" || echo "No changes to commit"
                            git push origin main
                        '''
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
                def pipelineStatus = currentBuild.result ?: 'SUCCESS'
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

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'founder@thebison.online',
                    from: 'hanumantn69@gmail.com',
                    replyTo: 'hanumantn69@gmail.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}
