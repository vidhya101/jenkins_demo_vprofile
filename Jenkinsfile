@Library("my_shared_library") _

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
                    echo 'Now Archiving this files .......'
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

        stage('CODE ANALYSIS with SONARQUBE') {
            steps {
                // Using shared library method for Sonar Analysis
                script {
                    sonarAnalysis(
                        projectKey: 'vprofile',
                        projectName: 'vprofile-repo',
                        projectVersion: '1.0',
                        sources: 'src/',
                        binaries: 'target/test-classes/com/visualpathit/account/controllerTest/',
                        junitReports: 'target/surefire-reports/',
                        jacocoReports: 'target/jacoco.exec',
                        checkstyleReports: 'target/checkstyle-result.xml'
                    )
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
                // Using shared library method for Nexus Upload
                script {
                    nexusUpload(
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
    }
    
    post {
        success {
            // Using shared library method for Success Email
            script {
                emailNotification.success()
            }
        }
        
        failure {
            // Using shared library method for Failure Email
            script {
                emailNotification.failure()
            }
        }
        
        always {
            // Using shared library method for Slack Notification
            script {
                slackNotification(channel: '#devops-vidhya-ci')
            }
        }
    }
}
