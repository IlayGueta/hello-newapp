def appName = "hello-newapp"
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
        )
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
                }
            )
        }

        stage('push') {
            container('docker') {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub',
                            usernameVariable: 'USERNAME',
                            passwordVariable: 'PASSWORD'
                        )
                    ]) {
                        echo "Deploying with username ${env.USERNAME}"

                        sh 'echo "$PASSWORD" | docker login -u "$USERNAME" --password-stdin'
                        sh "docker tag ${appName}:${env.BUILD_NUMBER} ${env.USERNAME}/${appName}:${env.BUILD_NUMBER}"
                        sh "docker push ${env.USERNAME}/${appName}:${env.BUILD_NUMBER}"
                    }
                }
            }
        } // end push
    }
}