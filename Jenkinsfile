def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

pipeline {
  agent {
    docker {
      image 'abhishekf5/maven-abhishek-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock' // mount Docker socket to access the host's Docker daemon
    }
  }
    
    tools {
        maven "MAVEN3.9"
        jdk "JDK17"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 2, unit: 'HOURS')
    }

    environment {
         // Explicitly set JAVA_HOME to the detected path
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        // Add JAVA_HOME to PATH
        PATH = "${JAVA_HOME}/bin:${PATH}"
        BUILD_TIMESTAMP = new Date().format('yyyy-MM-dd_HH-mm-ss')
        DOCKER_IMAGE_NAME = 'vprofile-app'
        DOCKER_IMAGE_TAG = "${DOCKER_IMAGE_NAME}:${BUILD_TIMESTAMP}-${BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS = credentials('docker-cred')
    }

    stages {
        stage('Fetch Code') {
            steps {
                git branch: 'atom', url: 'https://github.com/hkhcoder/vprofile-project.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('Unit Tests') {
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

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                always {
                    recordIssues(
                        enabledForFailure: true,
                        tools: [checkStyle(pattern: '**/target/checkstyle-result.xml')]
                    )
                }
            }
        }

        stage('SonarQube Analysis') {
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

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Prepare Dockerfile') {
            steps {
                script {
                    // Create a Dockerfile to package the WAR file
                    writeFile file: 'Dockerfile', text: '''
                    FROM tomcat:9-jdk17-openjdk-slim
                    COPY target/*.war /usr/local/tomcat/webapps/vprofile.war
                    EXPOSE 8080
                    CMD ["catalina.sh", "run"]
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        docker build -t ${DOCKER_IMAGE_TAG} .
                        docker tag ${DOCKER_IMAGE_TAG} ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_IMAGE_TAG}
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    sh """
                        echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin
                        docker push ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_IMAGE_TAG}
                        docker push ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Nexus Artifact Upload') {
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
                            <p><strong>Docker Image:</strong> ${DOCKER_IMAGE_TAG}</p>
                            <h3>Stages Summary:</h3>
                            <ul>
                                <li>Unit Tests: Completed</li>
                                <li>Checkstyle Analysis: Completed</li>
                                <li>SonarQube Analysis: Passed</li>
                                <li>Quality Gate: Passed</li>
                                <li>Docker Image: Built and Pushed</li>
                                <li>Artifact Upload: Successful</li>
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
                            <p><strong>Build URL:</strong> <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>
                            <p><strong>Failed Stage:</strong> ${currentBuild.result}</p>
                            <p>Check console output for error details.</p>
                        </body>
                    </html>
                """,
                to: '$DEFAULT_RECIPIENTS',
                mimeType: 'text/html',
                attachLog: true
            )
        }
        
        always {
            echo 'Sending Slack Notifications'
            slackSend channel: '#devops-cicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
            
            // Clean up Docker images to prevent disk space issues
            sh '''
                docker rmi ${DOCKER_IMAGE_TAG} || true
                docker rmi ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_IMAGE_TAG} || true
            '''
        }
    }
}
