pipeline {
    agent any

    tools {
        jdk 'jdk'
        maven 'maven'
    }

    environment{
        //here if you create any variable you will have global access, since it is environment no need of def
        packageVersion = ''
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'master', credentialsId: 'git-cred', url: 'https://github.com/kanalavinodkumar/Borardgame.git'
            }
        }

        stage('compile and test') {
            steps {
                sh '''
                mvn clean
                mvn compile
                mvn test
                '''
            }
        }

        stage('File system scan') {
            steps {
                sh "trivy fs --format table -o trivy-image-report.html ."
            }
        }

        stage('Sonarqube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Boardgame \
                        -Dsonar.projectkey=Boardgame \
                        -Dsonar.java.binaries=./var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/sonar-scanner/bin/sonar-scanner

                    '''
                }
            }

        }

        stage('Quality Gate') {
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'sonar' 
                }
            }
        }

        stage('Build') {
            steps {
                sh '''
                    mvn package
                '''
            }
        }

        stage('publish to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk', maven: 'maven', mavenSettingsConfig: '', traceability: true) {
                    sh 'mvn deploy'
                    
                }
            }
        }

        // stage('Docker Build') {
        //         steps {
        //             script{
        //                 sh """
        //                     docker build -t kanalavinodkumar/boardgame:latest .
        //                 """
        //             }
        //         }
        // }

        // stage('docker image scan') {
        //     steps {
        //         sh "trivy image --format table -o trivy-image-report.html kanalavinodkumar/boardgame"
        //     }
        // }

        // stage('Docker Push') {
        //         steps {
        //             script{
        //                 withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
        //                 sh 'docker push kanalavinodkumar/boardgame:latest'
        //                 }

        //             }
                    
        //         }
        // }

        // stage('Deployment') {
        //     steps {
        //         withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8', namespace: 'boardgame', restrictKubeConfigAccess: false, serverUrl: 'https://10.1.0.5:6443') {
        //             sh 'kubectl apply -f manifest.yaml'  
        //         }
        //     }
        // }
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
                to: 'kanalavinodkumar@gmail.com',
                from: 'vinodkumarkanala@gmail.com',
                replyTo: 'vinodkumarkanala@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}
}
