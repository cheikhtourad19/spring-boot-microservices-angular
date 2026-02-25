pipeline {

    agent any

    environment {

        GITHUB_REPO_URL    = 'https://github.com/cheikhtourad19/spring-boot-microservices-angular.git'
        GITHUB_BRANCH      = 'main'
        SONAR_SERVER_NAME  = 'SonarServer'
        SONAR_HOST_URL     = 'http://192.168.56.11:9000'
        SONAR_PROJECT_KEY  = 'spring-boot-microservices'
        SONAR_PROJECT_NAME = 'Spring Boot Microservices'

        COMPOSE_FILE       = 'docker-compose.yml'
        COMPOSE_PROJECT    = 'microservices'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()
        skipDefaultCheckout()
    }

    stages {

        stage('Checkout') {
            steps {
                sh """
                    if [ -d ".git" ]; then
                        echo "Repo exists — pulling latest changes..."
                        git remote set-url origin ${env.GITHUB_REPO_URL} 2>/dev/null || git remote add origin ${env.GITHUB_REPO_URL}
                        git fetch origin
                        git reset --hard origin/main
                        git clean -fd
                    else
                        echo "First run — cloning repository..."
                        git clone -b main ${env.GITHUB_REPO_URL} .
                    fi
                """
            }
        }

        stage('Build & Test') {
            steps {
                dir('backend') {
                    sh '''
                        mvn clean verify \
                            --batch-mode \
                            --no-transfer-progress \
                            -DskipTests=true
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(env.SONAR_SERVER_NAME) {
                    withCredentials([string(credentialsId: 'sonar_key', variable: 'SONAR_TOKEN')]) {
                        dir('backend') {
                            sh """
                                mvn sonar:sonar \
                                    --batch-mode \
                                    --no-transfer-progress \
                                    -Dsonar.host.url=${env.SONAR_HOST_URL} \
                                    -Dsonar.token=${SONAR_TOKEN} \
                                    -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \
                                    -Dsonar.projectName='${env.SONAR_PROJECT_NAME}' \
                                    -Dsonar.java.coveragePlugin=jacoco \
                                    -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                            """
                        }
                    }
                }
            }
        }


        stage('Quality Gate') {
            steps {
                echo "SonarQube analysis submitted. Check results at: ${env.SONAR_HOST_URL}/dashboard?id=${env.SONAR_PROJECT_KEY}"
            }
        }


        stage('Docker Compose Build') {
            steps {
                sh """
                    docker compose \
                        -p ${env.COMPOSE_PROJECT} \
                        -f ${env.WORKSPACE}/${env.COMPOSE_FILE} \
                        build --no-cache --parallel
                """
            }
        }

        stage('Docker Compose Up') {
            steps {
                sh """
                    docker compose \
                        -p ${env.COMPOSE_PROJECT} \
                        -f ${env.WORKSPACE}/${env.COMPOSE_FILE} \
                        up -d --remove-orphans
                """
                sh "docker compose -p ${env.COMPOSE_PROJECT} -f ${env.WORKSPACE}/${env.COMPOSE_FILE} ps"
            }
        }

    }

    // ─────────────────────────────────────────────────────────────────────────
    // Post-build actions
    // ─────────────────────────────────────────────────────────────────────────
    post {

        success {
            echo "Pipeline completed SUCCESSFULLY — Build #${env.BUILD_NUMBER}"
        }

        failure {
            echo "Pipeline FAILED — Build #${env.BUILD_NUMBER}"
        }

        unstable {
            echo 'Pipeline is UNSTABLE — tests may have failed. Check test reports.'
        }

        always {
            sh """
                docker compose \
                    -p ${env.COMPOSE_PROJECT} \
                    -f ${env.WORKSPACE}/${env.COMPOSE_FILE} \
                    down --volumes --remove-orphans || true
            """
        }
    }
}
