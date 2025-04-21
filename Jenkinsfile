pipeline {
    agent any
    parameters {
        string(name: 'DEPLOY_BRANCH', defaultValue: '', description: 'Branch to deploy for the changed service (e.g., dev_vets_service)')
    }
    environment {
        projectName = 'lab01hcmus'
        GITHUB_TOKEN = credentials('token-github')
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
        DOCKERHUB_REPO = 'ktei8htop15122004/clinic-microservices'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                // Fetch all branches to ensure DEPLOY_BRANCH is available
                sh 'git fetch origin'
            }
        }
        stage('Initialize') {
            steps {
                sh '''
                    java -version
                    mvn -version
                    docker -v
                '''
                // Verify Docker Hub login early to catch authentication issues
                sh '''
                    echo \$DOCKERHUB_CREDENTIALS_PSW | docker login -u \$DOCKERHUB_CREDENTIALS_USR --password-stdin || { echo "Docker Hub login failed. Check 'dockerhub' credentials in Jenkins."; exit 1; }
                '''
                step([
                    $class: "GitHubCommitStatusSetter",
                    reposSource: [$class: "ManuallyEnteredRepositorySource", url: "https://github.com/KhacThien88/developer_build_clinic"],
                    contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status"],
                    errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
                    statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: "START BUILD", state: "PENDING"]] ]
                ])
            }
        }
        stage('Developer Build (CD)') {
            when {
                expression { env.DEPLOY_BRANCH != '' }
            }
            steps {
                script {
                    def services = [
                        'admin-server',
                        'customers-service',
                        'vets-service',
                        'visits-service',
                        'genai-service',
                        'config-server',
                        'discovery-server',
                        'api-gateway'
                    ]

                    // Validate DEPLOY_BRANCH
                    def branchExists = sh(script: "git ls-remote --heads origin ${env.DEPLOY_BRANCH} | wc -l", returnStdout: true).trim() == '1'
                    if (!branchExists) {
                        error "Branch ${env.DEPLOY_BRANCH} does not exist in the repository."
                    }

                    // Checkout the DEPLOY_BRANCH
                    sh "git checkout ${env.DEPLOY_BRANCH}"

                    def changedModule = sh(script: "git diff --name-only origin/main...${env.DEPLOY_BRANCH} | grep -o 'spring-petclinic-[a-z-]*' | head -1", returnStdout: true).trim()
                    def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    def serviceName = changedModule ? changedModule.replace('spring-petclinic-', '') : ''

                    // Build and push the image for the specified branch if a module is changed
                    if (changedModule && env.DEPLOY_BRANCH) {
                        // Build the service's JAR if not already built
                        sh """
                            cd ${changedModule}
                            mvn clean install -DskipTests
                        """
                        // Define service-specific ports (verify these match server.port in each service's application.yml)
                        def portMap = [
                            'admin-server': '9090',
                            'customers-service': '8081',
                            'vets-service': '8083',
                            'visits-service': '8082',
                            'genai-service': '8084',
                            'config-server': '8888',
                            'discovery-server': '8761',
                            'api-gateway': '8080'
                        ]
                        def exposedPort = portMap[serviceName] ?: '9966'
                        sh """
                            mkdir -p docker/build
                            cp ${changedModule}/target/${changedModule}-3.4.1.jar docker/build/
                            cd docker/build
                            docker build -f ../Dockerfile -t ${DOCKERHUB_REPO}:${serviceName}-${commitId} \
                                --build-arg ARTIFACT_NAME=${changedModule}-3.4.1 \
                                --build-arg EXPOSED_PORT=${exposedPort} .
                            echo \$DOCKERHUB_CREDENTIALS_PSW | docker login -u \$DOCKERHUB_CREDENTIALS_USR --password-stdin
                            docker push ${DOCKERHUB_REPO}:${serviceName}-${commitId} || { echo "Failed to push ${DOCKERHUB_REPO}:${serviceName}-${commitId}. Check Docker Hub credentials and repository access."; exit 1; }
                            # Clean up temporary build context
                            rm -rf docker/build
                        """
                    }
                }
            }
        }
        stage('Cleanup') {
            when {
                expression { env.DEPLOY_BRANCH != '' }
            }
            steps {
                script {
                    def services = [
                        'admin-server',
                        'customers-service',
                        'vets-service',
                        'visits-service',
                        'genai-service',
                        'config-server',
                        'discovery-server',
                        'api-gateway'
                    ]
                    def changedModule = sh(script: "git diff --name-only origin/main...${env.DEPLOY_BRANCH} | grep -o 'spring-petclinic-[a-z-]*' | head -1", returnStdout: true).trim()
                    def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    def serviceName = changedModule ? changedModule.replace('spring-petclinic-', '') : ''

                    echo "Cleaning up deployed containers..."
                    services.each { svc ->
                        def imageTag = (svc == serviceName && env.DEPLOY_BRANCH && changedModule) ? "${svc}-${commitId}" : "${svc}-latest"
                        sh """
                            docker rm -f ${svc}-${imageTag} || true
                            echo "Cleaned up container for ${DOCKERHUB_REPO}:${imageTag}"
                        """
                    }
                }
            }
        }
    }
    post {
        success {
            step([
                $class: "GitHubCommitStatusSetter",
                reposSource: [$class: "ManuallyEnteredRepositorySource", url: "https://github.com/KhacThien88/developer_build_clinic"],
                contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status"],
                errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
                statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: "Build success", state: "SUCCESS"]] ]
            ])
            echo "Build and cleanup completed successfully. Log: ${env.BUILD_URL}"
        }
        failure {
            step([
                $class: "GitHubCommitStatusSetter",
                reposSource: [$class: "ManuallyEnteredRepositorySource", url: "https://github.com/KhacThien88/developer_build_clinic"],
                contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status"],
                errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
                statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: "Build failed", state: "FAILURE"]] ]
            ])
            echo "Build or cleanup failed. Check log: ${env.BUILD_URL}"
        }
        always {
            junit '**/target/surefire-reports/*.xml'
            archiveArtifacts artifacts: '**/target/site/jacoco/**', allowEmptyArchive: true
        }
    }
}