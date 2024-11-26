def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

pipeline {
    agent any

    tools {
        maven 'MAVEN3.9'
        jdk 'JDK17'
    }

    environment {
        JAVA_HOME = tool 'JDK17'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        DOCKER_IMAGE_TAG = "my-docker-image:${BUILD_NUMBER}"
        DOCKERHUB_USER = credentials('vidhya101')
        DOCKERHUB_PASS = credentials('RekhaGoel1_')
        SONARQUBE_SCANNER = tool 'sonar6.2'
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
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                always {
                    recordIssues(tools: [checkStyle(pattern: '**/target/checkstyle-result.xml')])
                }
            }
        }

        stage('Code Analysis with SonarQube') {
            environment {
                JAVA_OPTS = '-XX:+EnableDynamicAgentLoading --add-opens java.base/java.lang=ALL-UNNAMED'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                    sh """
                        ${SONARQUBE_SCANNER}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.junit.reportsPath=target/surefire-reports/
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
                writeFile file: 'Dockerfile', text: '''
                FROM tomcat:9-jdk17-openjdk-slim
                COPY target/*.war /usr/local/tomcat/webapps/vprofile.war
                EXPOSE 8080
                CMD ["catalina.sh", "run"]
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE_TAG} .
                    docker tag ${DOCKER_IMAGE_TAG} ${DOCKERHUB_USER}/${DOCKER_IMAGE_TAG}
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh """
                    echo ${DOCKERHUB_PASS} | docker login -u ${DOCKERHUB_USER} --password-stdin
                    docker push ${DOCKERHUB_USER}/${DOCKER_IMAGE_TAG}
                """
            }
        }

        stage('Upload Artifact') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '192.168.56.21:8081',
                    repository: 'vprofile-central-repo',
                    credentialsId: 'nexuslogin',
                    artifacts: [
                        [artifactId: 'vproapp', file: 'target/vprofile-v2.war', type: 'war']
                    ]
                )
            }
        }
    }

    post {
        success {
            emailext(
                subject: "✅ [SUCCESS] Pipeline: ${JOB_NAME} - Build #${BUILD_NUMBER}",
                body: """
                    <h2>Build Successful!</h2>
                    Pipeline: ${JOB_NAME}<br>
                    Build Number: ${BUILD_NUMBER}<br>
                    Build URL: <a href='${BUILD_URL}'>${BUILD_URL}</a><br>
                    <h3>Stages Summary:</h3>
                    - Unit Tests: Completed<br>
                    - Checkstyle Analysis: Completed<br>
                    - SonarQube Analysis: Passed<br>
                    - Quality Gate: Passed<br>
                """,
                mimeType: 'text/html',
                to: '$DEFAULT_RECIPIENTS'
            )
        }

        failure {
            emailext(
                subject: "❌ [FAILED] Pipeline: ${JOB_NAME} - Build #${BUILD_NUMBER}",
                body: """
                    <h2 style='color: red;'>Build Failed!</h2>
                    Pipeline: ${JOB_NAME}<br>
                    Build Number: ${BUILD_NUMBER}<br>
                    Failed Stage: ${currentBuild.result}<br>
                    Check logs for details.<br>
                """,
                mimeType: 'text/html',
                to: '$DEFAULT_RECIPIENTS'
            )
        }

        always {
            slackSend(
                channel: '#devops-vidhya-ci',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "${currentBuild.currentResult}: Job ${JOB_NAME} Build #${BUILD_NUMBER}"
            )

            sh """
                docker rmi ${DOCKER_IMAGE_TAG} || true
                docker rmi ${DOCKERHUB_USER}/${DOCKER_IMAGE_TAG} || true
            """
        }
    }
}
