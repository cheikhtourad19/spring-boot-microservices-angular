pipeline {

    agent any

    // ─────────────────────────────────────────────────────────────────────────
    // Pipeline-wide environment variables
    // Sensitive values (SONAR_TOKEN, etc.) are stored as Jenkins Credentials:
    //   Manage Jenkins → Credentials → Global → Add Credential (Secret text)
    // ─────────────────────────────────────────────────────────────────────────
    environment {
        // GitHub repository
        GITHUB_REPO_URL    = 'https://github.com/cheikhtourad19/spring-boot-microservices-angular.git'
        GITHUB_BRANCH      = 'main'

        // SonarQube – matches the server name configured in
        //   Manage Jenkins → Configure System → SonarQube servers
        SONAR_SERVER_NAME  = 'SonarServer'                   // Name field in Jenkins
        SONAR_HOST_URL     = 'http://192.168.56.11:9000'     // Server URL field
        // Authentication token is stored as Jenkins credential ID: sonar_key
        SONAR_PROJECT_KEY  = 'spring-boot-microservices'
        SONAR_PROJECT_NAME = 'Spring Boot Microservices'

        // Docker Compose file (relative to workspace root)
        COMPOSE_FILE       = 'docker-compose.yml'

        // Compose project name (keeps container names stable across builds)
        COMPOSE_PROJECT    = 'microservices'
    }

    // ─────────────────────────────────────────────────────────────────────────
    // Build options
    // ─────────────────────────────────────────────────────────────────────────
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))   // keep last 10 builds
        timeout(time: 60, unit: 'MINUTES')               // fail-safe overall timeout
        disableConcurrentBuilds()                         // one build at a time
    }

    // ─────────────────────────────────────────────────────────────────────────
    // Stages
    // ─────────────────────────────────────────────────────────────────────────
    stages {

        // ── 1. Build & Test (all backend modules via parent POM) ─────────────
        stage('Build & Test') {
            steps {
                dir('backend') {
                    sh '''
                        mvn clean verify \
                            --batch-mode \
                            --no-transfer-progress \
                            -DskipTests=false
                    '''
                }
            }
        }

        // ── 3. SonarQube Analysis ─────────────────────────────────────────────
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

        // ── 4. Quality Gate ───────────────────────────────────────────────────
        stage('Quality Gate') {
            steps {
                echo 'Waiting for SonarQube Quality Gate result …'
                // Waits up to 5 min for the webhook callback from SonarQube.
                // Configure the webhook in SonarQube → Administration → Webhooks:
                //   URL: http://<jenkins-host>/sonarqube-webhook/
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ── 5. Docker Compose Build ───────────────────────────────────────────
        stage('Docker Compose Build') {
            steps {
                sh """
                    docker-compose \
                        -p ${env.COMPOSE_PROJECT} \
                        -f ${env.COMPOSE_FILE} \
                        build --no-cache --parallel
                """
            }
        }

        // ── 6. Docker Compose Up ──────────────────────────────────────────────
        stage('Docker Compose Up') {
            steps {
                sh """
                    docker-compose \
                        -p ${env.COMPOSE_PROJECT} \
                        -f ${env.COMPOSE_FILE} \
                        up -d --remove-orphans
                """
                sh "docker-compose -p ${env.COMPOSE_PROJECT} -f ${env.COMPOSE_FILE} ps"
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
            sh """
                docker-compose \
                    -p ${env.COMPOSE_PROJECT} \
                    -f ${env.COMPOSE_FILE} \
                    down --volumes --remove-orphans || true
            """
        }

        unstable {
            echo 'Pipeline is UNSTABLE — tests may have failed. Check test reports.'
        }

        always {
            deleteDir()
        }
    }
}
