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
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
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
                    archiveArtifacts artifacts: '**/target/*.war', allowEmptyArchive: false
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
                    docker tag ${DOCKER_IMAGE_TAG} ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_IMAGE_TAG}
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                    sh """
                        echo ${DOCKERHUB_PASS} | docker login -u ${DOCKERHUB_USER} --password-stdin
                        docker push ${DOCKERHUB_USER}/${DOCKER_IMAGE_TAG}
                    """
                }
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
                subject: "✅ [SUCCESS] Pipeline: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    <html>
                        <body>
                            <h2>Build Successful!</h2>
                            <p><strong>Pipeline:</strong> ${env.JOB_NAME}</p>
                            <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                            <p><strong>Build URL:</strong> <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>
                            <p>Check console output for details.</p>
                        </body>
                    </html>
                """,
                to: '$DEFAULT_RECIPIENTS',
                mimeType: 'text/html',
                attachLog: true
            )
        }

        failure {
            emailext(
                subject: "❌ [FAILED] Pipeline: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    <html>
                        <body>
                            <h2>Build Failed!</h2>
                            <p><strong>Pipeline:</strong> ${env.JOB_NAME}</p>
                            <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                            <p><strong>Build URL:</strong> <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>
                        </body>
                    </html>
                """,
                to: '$DEFAULT_RECIPIENTS',
                mimeType: 'text/html',
                attachLog: true
            )
        }
    }
}
