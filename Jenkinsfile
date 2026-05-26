pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'hxjerry/teedy'
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        DOCKER_CREDENTIALS_ID = 'dockerhub'
        DOCKER_SERVICE = 'teedy'
        DOCKER_REPLICAS = '3'
    }
    options {
        // Skips the automatic checkout at the start
        skipDefaultCheckout()
    }
    stages {
        stage('Clean Workspace') {
            steps {
                // Deletes the workspace before the build starts
                cleanWs()
                checkout scm
            }
        }

        stage('Clean') {
            steps {
                sh 'mvn -B clean'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn -B compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -B test -Dmaven.test.failure.ignore=true'
            }
        }

        stage('Package') {
            steps {
                sh 'mvn -B -U install -DskipTests'
            }
        }

        stage('PMD') {
            steps {
                sh 'mvn -B -U pmd:pmd'
            }
        }

        stage('JaCoCo') {
            steps {
                sh 'mvn -B jacoco:report'
            }
        }

        stage('Javadoc') {
            steps {
                sh 'mvn -B javadoc:javadoc -Ddoclint=none'
            }
        }

        stage('Site') {
            steps {
                sh 'mvn -B -U site -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build \
                        -t "$DOCKER_IMAGE:$DOCKER_TAG" \
                        -t "$DOCKER_IMAGE:latest" \
                        .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh '''
                        set -eu
                        printf '%s' "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker push "$DOCKER_IMAGE:$DOCKER_TAG"
                        docker push "$DOCKER_IMAGE:latest"
                    '''
                }
            }
        }

        stage('Deploy Docker Service') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh '''
                        set -eu
                        printf '%s' "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

                        if [ "$(docker info --format '{{.Swarm.LocalNodeState}}')" = "inactive" ]; then
                            docker swarm init
                        fi

                        if docker service inspect "$DOCKER_SERVICE" >/dev/null 2>&1; then
                            docker service update \
                                --with-registry-auth \
                                --image "$DOCKER_IMAGE:$DOCKER_TAG" \
                                --replicas "$DOCKER_REPLICAS" \
                                "$DOCKER_SERVICE"
                        else
                            docker service create \
                                --with-registry-auth \
                                --name "$DOCKER_SERVICE" \
                                --replicas "$DOCKER_REPLICAS" \
                                --publish published=8080,target=8080 \
                                "$DOCKER_IMAGE:$DOCKER_TAG"
                        fi
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            archiveArtifacts artifacts: '**/target/site/**/*.*', fingerprint: true
            archiveArtifacts artifacts: '**/target/**/*.jar', fingerprint: true
            archiveArtifacts artifacts: '**/target/**/*.war', fingerprint: true
            junit '**/target/surefire-reports/*.xml'
        }
    }
}
