pipeline {
    agent { label 'Jenkins-Agent' }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "gaurav5213"
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }

    stages {

        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/gauravmishra52/registration_app'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            steps {
                sh "mvn test"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'jenkins-sonarcube-tokens') {
                        sh "mvn sonar:sonar"
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarcube-tokens'
                }
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker build -t $IMAGE_NAME:$BUILD_NUMBER .
                            docker tag $IMAGE_NAME:$BUILD_NUMBER $IMAGE_NAME:latest
                            docker push $IMAGE_NAME:$BUILD_NUMBER
                            docker push $IMAGE_NAME:latest
                        '''
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy image ${IMAGE_NAME}:latest \
                    --no-progress --scanners vuln --exit-code 0 \
                    --severity HIGH,CRITICAL --format table
                '''
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                sh '''
                    docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
                    docker rmi ${IMAGE_NAME}:latest || true
                '''
            }
        }
    stage("Trigger CD Pipeline") {
    steps {
        withCredentials([usernamePassword(credentialsId: 'jenkins-api-token', usernameVariable: 'CD_USER', passwordVariable: 'CD_TOKEN')]) {
            sh '''
                echo "🚀 Triggering CD Pipeline..."
                RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" -k --user $CD_USER:$CD_TOKEN -X POST \
                    -H "cache-control: no-cache" \
                    -H "content-type: application/x-www-form-urlencoded" \
                    --data "IMAGE_TAG=${IMAGE_TAG}" \
                    http://34.240.240.74:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token)

                echo "HTTP response: $RESPONSE"
                if [ "$RESPONSE" -ne 201 ] && [ "$RESPONSE" -ne 200 ]; then
                    echo "❌ CD trigger failed!"
                    exit 1
                fi
                echo "✅ CD Pipeline triggered successfully."
            '''
        }
    }
}
 post {
        failure {
            emailext(
                body: '''${SCRIPT, template="groovy-html.template"}''',
                subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ❌ Failed",
                mimeType: 'text/html',
                to: "amank844932@gmail.com"
            )
        }

        success {
            emailext(
                body: '''${SCRIPT, template="groovy-html.template"}''',
                subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ✅ Successful",
                mimeType: 'text/html',
                to: "amank844932@gmail.com"
            )
        }
    }
}
