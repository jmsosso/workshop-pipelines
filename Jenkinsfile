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
        APP_NAME = "juanmi-app"
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

        stage('Build Docker image') {
            steps {
                echo "Build Docker image ***************************************"
                sh "./mvnw docker:build"
            }
        }

        stage('Run Docker image') {
            steps {
                echo "Run Docker image *****************************************"
                sh "docker run --name ${TEST_CONTAINER_NAME} --detach --rm --network ci --expose ${APP_LISTENING_PORT} --expose 6300 --env JAVA_OPTS='-Dserver.port=${APP_LISTENING_PORT} -Dspring.profiles.active=ci -javaagent:/jacocoagent.jar=output=tcpserver,address=*,port=6300' ${ORG_NAME}/${APP_NAME}:latest"
            }
        }

        stage('Integration tests') {
            steps {
                echo "Execute integration tests ********************************"
                sh "curl --retry 5 --retry-connrefused --connect-timeout 5 --max-time 5 http://${TEST_CONTAINER_NAME}:${APP_LISTENING_PORT}/${APP_CONTEXT_ROOT}/actuator/health"
                sh "./mvnw failsafe:integration-test failsafe:verify -DargLine=\"-Dtest.target.server.url=http://${TEST_CONTAINER_NAME}:${APP_LISTENING_PORT}/${APP_CONTEXT_ROOT}\""
                sh "java -jar target/dependency/jacococli.jar dump --address ${TEST_CONTAINER_NAME} --port 6300 --destfile target/jacoco-it.exec"
                junit 'target/failsafe-reports/*.xml'
                jacoco execPattern: 'target/jacoco-it.exec'
            }
        }

        stage('Performance tests') {
            steps {
                echo "Execute performance tests ********************************"
                sh "./mvnw jmeter:jmeter jmeter:results -Djmeter.target.host=${TEST_CONTAINER_NAME} -Djmeter.target.port=${APP_LISTENING_PORT} -Djmeter.target.root=${APP_CONTEXT_ROOT}"
                perfReport sourceDataFiles: 'target/jmeter/results/*.csv'
            }
        }

    }
}