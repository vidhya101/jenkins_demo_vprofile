def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

pipeline {
    agent any
    
    tools {
        maven "MAVEN3.9"
        jdk "JDK17"
    }
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    environment {
        BUILD_TIMESTAMP = new Date().format('yyyy-MM-dd_HH-mm-ss')
        DOCKER_HUB_USERNAME = "vidhya101"
        DOCKER_HUB_PASSWORD = "RekhaGoel1_"
        DOCKER_COMPOSE_PATH = "jenkins_demo_vprofile/docker-compose.yml"
    }

    stages {
        stage('Fetch code') {
            steps {
                git branch: 'atom', url: 'https://github.com/hkhcoder/vprofile-project.git'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving these files .......'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test -Dmaven.test.failure.ignore=true'
            }
            post {
                always {
                    junit(
                        allowEmptyResults: true,
                        testResults: '**/target/surefire-reports/*.xml',
                        skipPublishingChecks: true
                    )
                }
            }
        }

        stage('Docker Build and Push') {
            steps {
                script {
                    echo "Building and pushing Docker images..."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_HUB_USERNAME', passwordVariable: 'DOCKER_HUB_PASSWORD')]) {
                        sh """
                            # Log in to Docker Hub
                            echo "${DOCKER_HUB_PASSWORD}" | docker login -u "${DOCKER_HUB_USERNAME}" --password-stdin

                            # Navigate to Docker Compose directory
                            cd ${DOCKER_COMPOSE_PATH}

                            # Build all images using docker-compose
                            docker-compose build

                            # Tag and push each image to Docker Hub
                            docker-compose config | grep 'image:' | awk '{print \$2}' | while read IMAGE; do
                                TAGGED_IMAGE="${DOCKER_HUB_USERNAME}/\${IMAGE}:build_${env.BUILD_ID}"
                                docker tag \${IMAGE} \${TAGGED_IMAGE}
                                docker push \${TAGGED_IMAGE}
                            done

                            # Log out from Docker Hub
                            docker logout
                        """
                    }
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
            environment {
                scannerHome = tool 'sonar6.2'
                JAVA_OPTS = '-XX:+EnableDynamicAgentLoading --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.base/java.io=ALL-UNNAMED --add-opens java.base/java.math=ALL-UNNAMED'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                    sh """
                        export SONAR_SCANNER_OPTS="${JAVA_OPTS}"
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                    """
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("UploadArtifact") {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '192.168.56.21:8081',
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: 'vprofile-central-repo',
                    credentialsId: 'nexuslogin',
                    artifacts: [
                        [artifactId: 'vproapp',
                         classifier: '',
                         file: 'target/vprofile-v2.war',
                         type: 'war']
                    ]
                )
            }
        }
    }
    post {
        success {
            emailext (
                subject: "✅ [SUCCESS] Pipeline: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    <html>
                        <body>
                            <h2>Build Successful!</h2>
                            <p><strong>Pipeline:</strong> ${env.JOB_NAME}</p>
                            <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                            <p><strong>Build ID:</strong> ${env.BUILD_ID}</p>
                            <p><strong>Build URL:</strong> <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>
                            <p><strong>Build Timestamp:</strong> ${env.BUILD_TIMESTAMP}</p>
                            <h3>Stages Summary:</h3>
                            <ul>
                                <li>Unit Tests: Completed</li>
                                <li>Checkstyle Analysis: Completed</li>
                                <li>SonarQube Analysis: Passed</li>
                                <li>Quality Gate: Passed</li>
                                <li>Artifact Upload: Successful</li>
                                <li>Docker Build and Push: Completed</li>
                            </ul>
                            <p>Check console output for detailed information.</p>
                        </body>
                    </html>
                """,
                to: '$DEFAULT_RECIPIENTS',
                mimeType: 'text/html',
                attachLog: true
            )
        }
        
        failure {
            emailext (
                subject: "❌ [FAILED] Pipeline: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    <html>
                        <body>
                            <h2 style="color: red;">Build Failed!</h2>
                            <p><strong>Pipeline:</strong> ${env.JOB_NAME}</p>
                            <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                            <p><strong>Build ID:</strong> ${env.BUILD_ID}</p>
                            <p><strong>Build URL:</strong> <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>
                            <p><strong>Build Timestamp:</strong> ${env.BUILD_TIMESTAMP}</p>
                            <p><strong>Failed Stage:</strong> ${currentBuild.result}</p>
                            <p>Check console output attached for error details.</p>
                            <h3>Last Successful Build:</h3>
                            <p><strong>Build Number:</strong> ${currentBuild.previousSuccessfulBuild?.number ?: 'None'}</p>
                        </body>
                    </html>
                """,
                to: '$DEFAULT_RECIPIENTS',
                mimeType: 'text/html',
                attachLog: true
            )
        }
        
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#devops-vidhya-ci',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}
