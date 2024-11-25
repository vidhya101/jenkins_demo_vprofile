def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17-alpine'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
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
        // Explicitly set JAVA_HOME and PATH
        JAVA_HOME = tool 'JDK17'
        PATH = "${JAVA_HOME}/bin:${PATH}"
        BUILD_TIMESTAMP = new Date().format('yyyy-MM-dd_HH-mm-ss')
    }
    
    stages {
        stage('System Check') {
            steps {
                script {
                    try {
                        sh '''
                            echo "System Diagnostics:"
                            echo "-------------------"
                            echo "Current User: $(whoami)"
                            echo "HOME Directory: $HOME"
                            echo "JAVA_HOME: $JAVA_HOME"
                            echo "PATH: $PATH"
                            
                            which java
                            java -version
                            which mvn
                            mvn --version
                            
                            echo "Directory Contents of JAVA_HOME:"
                            ls -la $JAVA_HOME
                        '''
                    } catch (Exception e) {
                        echo "Diagnostic check failed: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/vidhya101/jenkins_demo_vprofile.git'
            }
        }
        
        stage('Build') {
            steps {
                script {
                    try {
                        sh '''
                            echo "Build Environment:"
                            echo "JAVA_HOME: ${JAVA_HOME}"
                            echo "PATH: ${PATH}"
                            
                            # Clean and compile the project
                            mvn clean compile -X
                        '''
                    } catch (Exception e) {
                        echo "Build failed: ${e.getMessage()}"
                        throw e
                    }
                }
            }
            post {
                success {
                    echo 'Build completed successfully'
                }
                failure {
                    echo 'Build failed'
                }
            }
        }

        stage('Unit Test') {
            steps {
                script {
                    try {
                        sh '''
                            # Run unit tests with detailed output
                            mvn test -Dmaven.test.failure.ignore=true -X
                        '''
                    } catch (Exception e) {
                        echo "Unit Test stage failed: ${e.getMessage()}"
                        throw e
                    }
                }
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
                script {
                    try {
                        sh 'mvn checkstyle:checkstyle -X'
                    } catch (Exception e) {
                        echo "Checkstyle Analysis failed: ${e.getMessage()}"
                        // Uncomment the next line if you want to fail the build on checkstyle issues
                        // throw e
                    }
                }
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

        stage('SonarQube Code Analysis') {
            environment {
                scannerHome = tool 'sonar6.2'
                JAVA_OPTS = '-XX:+EnableDynamicAgentLoading --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.base/java.io=ALL-UNNAMED --add-opens java.base/java.math=ALL-UNNAMED'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                    script {
                        try {
                            sh """
                                export SONAR_SCANNER_OPTS="${JAVA_OPTS}"
                                ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=vprofile \
                                -Dsonar.projectName=vprofile-repo \
                                -Dsonar.projectVersion=${BUILD_NUMBER} \
                                -Dsonar.sources=src/ \
                                -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                                -Dsonar.junit.reportsPath=target/surefire-reports/ \
                                -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                                -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                            """
                        } catch (Exception e) {
                            echo "SonarQube Analysis failed: ${e.getMessage()}"
                            throw e
                        }
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build and Push Docker Image') {
            environment {
                DOCKER_IMAGE = "vidhya101/ultimate-cicd:${BUILD_NUMBER}"
                REGISTRY_CREDENTIALS = credentials('docker-cred')
            }
            steps {
                script {
                    try {
                        sh "docker build -t ${DOCKER_IMAGE} ."
                        docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                            def dockerImage = docker.image("${DOCKER_IMAGE}")
                            dockerImage.push()
                            dockerImage.push('latest')
                        }
                    } catch (Exception e) {
                        echo "Docker build and push failed: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }

        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = "jenkins_demo_vprofile"
                GIT_USER_NAME = "vidhya101"
            }
            steps {
                script {
                    try {
                        withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                            sh '''
                                git config --global user.email "vidhyashankargoel1996@gmail.com"
                                git config --global user.name "vidhya101"
                                git clone https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git
                                cd ${GIT_REPO_NAME}
                                sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" kubedefs/appdeploy.yaml
                                git add kubedefs/appdeploy.yaml
                                git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                                git push origin HEAD:main
                            '''
                        }
                    } catch (Exception e) {
                        echo "Deployment file update failed: ${e.getMessage()}"
                        throw e
                    }
                }
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
            script {
                def buildStatus = currentBuild.currentResult
                def color = COLOR_MAP.containsKey(buildStatus) ? COLOR_MAP[buildStatus] : 'warning'
                
                slackSend(
                    channel: '#devops-vidhya-ci',
                    color: color,
                    message: """
*${buildStatus}:* Job ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Duration: ${currentBuild.duration/1000} seconds
More info: ${env.BUILD_URL}
                    """
                )
            }
        }
    }
}
