pipeline {
    agent any
    
    tools {
        maven 'maven'
    }
    
    parameters {
        choice(name:'DEPLOY_ENV', choices: ['blue','green'], description:'Choose which env to deploy: Blue or Green')
        choice(name:'DOCKER_TAG', choices: ['blue','green'], description:'Choose docker img for deployment')
        booleanParam(name:'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between blue and green')
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = "kanalavinodkumar/boardgame"
        TAG = "${params.DOCKER_TAG}"
        KUBE_NAMESPACE = 'boardgame'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git credentialsId: 'jenkins-aws-github-creds', url: 'https://github.com/kanalavinodkumar/Borardgame.git'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('Test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        stage('Trivy FS scan') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=Boardgame -Dsonar.projectName=Boardgame -Dsonar.java.binaries=target"
                }
            }
        }
        //stage('Quality Gate check') {
        //    steps {
        //        timeout(time: 1, unit: 'HOURS') {
        //            waitForQualityGate abortPipeline: false
        //        }
        //    }
        //}
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        stage('Publish to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global', jdk: '', maven: 'maven', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }
        
        stage('Docker build & Tag image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                    }
                }
            }
        }
        stage('Docker Push image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }
        stage('Deploy Service') {
            steps {
                script{
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-token', namespace: 'boardgame', restrictKubeConfigAccess: false, serverUrl: 'https://api.vinodhub.online') {
                    sh """ ls -la
                            pwd
                        if ! kubectl get svc boardgame-svc -n ${KUBE_NAMESPACE}; then
                        kubectl apply -f service.yaml -n ${KUBE_NAMESPACE} --validate=false
                        fi
                    """
                    }
                }
                
            }
        }
        stage('Deploy Pods') {
            steps {
                script{
                    def deploymentFile = ""
                    if (params.DEPLOY_ENV == 'blue') {
                        deploymentFile = 'blue.yaml'
                    } else {
                        deploymentFile = 'green.yaml'
                    }
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-token', namespace: 'boardgame', restrictKubeConfigAccess: false, serverUrl: 'https://api.vinodhub.online') {
                    sh 'kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE} --validate=false'
                    
                    }
                }
                
            }
        }   
        stage('Switch traffic') {
            when {
                expression {return params.SWITCH_TRAFFIC }
            }
            steps {
                script{
                    def newEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-token', namespace: 'boardgame', restrictKubeConfigAccess: false, serverUrl: 'https://api.vinodhub.online') {
                    sh '''kubectl patch service boardgame-svc -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"boardgame\\", \\"version\\":\\"''' + newEnv + '''\\"}}}" -n ${KUBE_NAMESPACE} --validate=false
                    '''

                    }
                    echo "Traffic has been switched to ${newEnv} environment"
                }
                
            }
        }
        stage('Verify Deployment') {
            steps {
                script{
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-token', namespace: 'boardgame', restrictKubeConfigAccess: false, serverUrl: 'https://api.vinodhub.online') {
                    sh """
                        kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                        kubectl get svc -n ${KUBE_NAMESPACE}
                    """
                    }
                }
                
            }
        }
    }
    post{
        always{
            echo 'cleaning up workspace'
            }
    }
//     post {
//     always {
//         script {
//             def jobName = env.JOB_NAME
//             def buildNumber = env.BUILD_NUMBER
//             def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
//             def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

//             def body = """
//                 <html>
//                 <body>
//                 <div style="border: 4px solid ${bannerColor}; padding: 10px;">
//                 <h2>${jobName} - Build ${buildNumber}</h2>
//                 <div style="background-color: ${bannerColor}; padding: 10px;">
//                 <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
//                 </div>
//                 <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
//                 </div>
//                 </body>
//                 </html>
//             """

//             emailext (
//                 subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
//                 body: body,
//                 to: 'kanalavinodkumar@gmail.com',
//                 from: 'vinodkumarkanala@gmail.com',
//                 replyTo: 'vinodkumarkanala@gmail.com',
//                 mimeType: 'text/html',
//                 attachmentsPattern: 'trivy-image-report.html'
//             )
//         }
//     }
// }
}
