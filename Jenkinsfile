def appName = "ilay-infrastructure-api"
def repo = "ilagueta"  // Replace with your DockerHub username
def appimage = "${repo}/${appName}"
def apptag = "${env.BUILD_NUMBER}"


podTemplate(
    containers: [
        containerTemplate(
            name: 'jnlp',
            image: 'jenkins/inbound-agent',
            ttyEnabled: true
        ),
        containerTemplate(
            name: 'docker',
            image: 'docker:26-dind',
            privileged: true,
            args: '--storage-driver=vfs --host=tcp://0.0.0.0:2375'
        ),
        containerTemplate(
            name: 'trivy',
            image: 'aquasec/trivy:latest',
            command: 'cat',
            ttyEnabled: true
        ),
        containerTemplate(
            name: 'helm',
            image: 'alpine/helm:3.17.3',
            command: 'cat',
            ttyEnabled: true
        ),
        containerTemplate(
            name: 'sonar',
            image: 'sonarsource/sonar-scanner-cli:latest',
            command: 'cat',
            ttyEnabled: true
        ),
    ],
    volumes: [
        emptyDirVolume(mountPath: '/var/run', memory: false)
    ]
) {
    node(POD_LABEL) {
        stage('chackout') {
            container('jnlp') {
                sh '/usr/bin/git config --global http.sslVerify false'
                checkout scm
            }
        } // end chackout

        stage('Build & Code Quality') {
            parallel(
                'Build Docker Image': {
                    container('docker') {
                        echo "Building docker image..."
                        sh "docker build -t ${appName}:${env.BUILD_NUMBER} ."
                    }
                },
                'Trivy Scan': {
                    container('trivy') {
                        echo "Running Trivy scan..."
                        sh "trivy fs ."
                    }
                },
                'SonarQube Scan': {
                    container('sonar') {
                        withCredentials([
                            string(
                                credentialsId: 'sonar-token',
                                variable: 'SONAR_TOKEN'
                            )
                        ]) {

                            withEnv([
                                'SONAR_HOST_URL=http://host.docker.internal:9000'
                            ]) {
                                echo "Running SonarQube analysis..."

                                sh '''
                                sonar-scanner \
                                    -Dsonar.projectKey=infrastructure-api-ci-pipeline \
                                    -Dsonar.sources=.
                                '''    
                            }
                        }
                    }
                }
            )
        }
        stage('Push') {
            container('docker') {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'USERNAME',
                        passwordVariable: 'PASSWORD'
                    )
                ]) {
                    echo "Pushing image with username ${env.USERNAME}"

                    sh 'echo "$PASSWORD" | docker login -u "$USERNAME" --password-stdin'
                    sh "docker tag ${appName}:${env.BUILD_NUMBER} ${env.USERNAME}/${appName}:${env.BUILD_NUMBER}"
                    sh "docker push ${env.USERNAME}/${appName}:${env.BUILD_NUMBER}"
                }        
            } // end push
        }

        stage('Deploy') {
            container('helm') {
                echo "Deploying application with Helm..."
                
                sh "helm template hello-newapp ./chart > hello-newapp.yaml"
            }
        }
    }
}