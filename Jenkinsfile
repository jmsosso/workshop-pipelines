#!groovy

pipeline {
    agent {
        docker {
            image 'adoptopenjdk/openjdk11:jdk-11.0.3_7'
            args '--network ci'
        }
    }

    environment {
        ORG_NAME = "jmsosso"
        APP_NAME = "workshop-pipelines"
        APP_CONTEXT_ROOT = "/"
        APP_LISTENING_PORT = "8080"
        TEST_CONTAINER_NAME = "ci-${APP_NAME}-${BUILD_NUMBER}"
        DOCKER_HUB = credentials("${ORG_NAME}-docker-hub")
    }

    stages {
        stage('Compile') {
            steps {
                echo 'Compiling ************************************************'
                sh './mvnw clean compile'
            }
        }

        stage('Unit tests') {
            steps {
                echo "Execute unit tests ***************************************"
                sh "./mvnw test"
                junit 'target/surefire-reports/*.xml'
                jacoco execPattern: 'target/jacoco.exec'
            }
        }

        stage('Mutation tests') {
            steps {
                echo "Execute mutation tests ***********************************"
                sh "./mvnw org.pitest:pitest-maven:mutationCoverage"
            }
        }

        stage('Package') {
            steps {
                echo "Packaging project ****************************************"
                sh "./mvnw package -DskipTests"
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }
}