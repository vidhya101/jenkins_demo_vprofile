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
        DOCKER_IMAGE_TAG = "my-docker-image:${env.BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS_USR = credentials('vidhya101')
        DOCKERHUB_CREDENTIALS_PSW = credentials('RekhaGoel1_')
        SONARQUBE_SCANNER = tool 'sonar6.2'
        SONARQUBE_SERVER = 'sonarserver'
    }

    stages {
        stage('Fetch Code') {
            steps {
                git branch: 'atom', url: 'https://github.com/hkhcoder/vprofile-project.git'
            }
        }

        stage('Build') {
            steps {
                script {
                    echo "Using JAVA_HOME: ${JAVA_HOME}"
                    sh 'mvn clean install -DskipTests'
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('Unit Tests') {
            steps {
                script {
                    echo "Running Unit Tests..."
                    sh 'mvn test -Dmaven.test.failure.ignore=true'
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
                    echo "Running Checkstyle..."
                    sh 'mvn checkstyle:checkstyle'
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

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv(SONARQUBE_SERVER) {
                        echo "Running SonarQube Analysis..."
                        sh """
                            export SONAR_SCANNER_OPTS="-XX:+EnableDynamicAgentLoading --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED"
                            ${SONARQUBE_SCANNER}/bin/sonar-scanner \
                            -Dsonar.projectKey=vprofile \
                            -Dsonar.sources=src/ \
                            -Dsonar.java.binaries=target/ \
                            -Dsonar.junit.reportsPath=target/surefire-reports/ \
                            -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                        """
                    }
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
                    """
                }
            }
        }

        stage('Nexus Artifact Upload') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: 'http://192.168.56.21:8081',
                    groupId: 'QA',
                    version: "${env.BUILD_NUMBER}",
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
                subject: "✅ [SUCCESS] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build succeeded: ${env.BUILD_URL}",
                to: '$DEFAULT_RECIPIENTS'
            )
        }

        failure {
            emailext (
                subject: "❌ [FAILURE] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build failed: ${env.BUILD_URL}",
                to: '$DEFAULT_RECIPIENTS'
            )
        }

        always {
            slackSend channel: '#devops-cicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} #${env.BUILD_NUMBER} - More info: ${env.BUILD_URL}"

            sh '''
                docker rmi ${DOCKER_IMAGE_TAG} || true
                docker rmi ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_IMAGE_TAG} || true
            '''
        }
    }
}
