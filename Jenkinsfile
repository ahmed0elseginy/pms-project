pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
        nodejs 'Node22'
    }

    environment {
        DOCKER_COMPOSE = 'docker compose'
        DOCKER_REGISTRY = 'ghcr.io/ahmed0elseginy'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    extensions: [
                        [$class: 'SubmoduleOption', disableSubmodules: false, parentCredentials: true, recursiveSubmodules: true, reference: '', trackingSubmodules: false]
                    ],
                    userRemoteConfigs: [[credentialsId: 'git-credentials', url: 'https://github.com/ahmed0elseginy/pms-project.git']]
                ])
            }
        }

        stage('Test Backend') {
            steps {
                dir('pms-backend') {
                    sh 'mvn -Pdefault -pl main/main-execution -am clean verify'
                }
            }
        }

        stage('Test Frontend') {
            steps {
                dir('pms-frontend') {
                    sh 'npm ci && npm run build'
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '${DOCKER_COMPOSE} build'
            }
        }

        stage('Docker Push') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'git-credentials', usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_PASS')]) {
                    sh """
                        docker logout ${DOCKER_REGISTRY} 2>/dev/null || true
                        echo \$REGISTRY_PASS | docker login ${DOCKER_REGISTRY} -u \$REGISTRY_USER --password-stdin

                        docker tag pms-backend-app:local ${DOCKER_REGISTRY}/pms-backend-app:${env.BUILD_NUMBER}
                        docker tag pms-frontend-app:local ${DOCKER_REGISTRY}/pms-frontend-app:${env.BUILD_NUMBER}
                        docker push ${DOCKER_REGISTRY}/pms-backend-app:${env.BUILD_NUMBER}
                        docker push ${DOCKER_REGISTRY}/pms-frontend-app:${env.BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '${DOCKER_COMPOSE} down'
                sh '${DOCKER_COMPOSE} up -d'
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
            cleanWs()
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}