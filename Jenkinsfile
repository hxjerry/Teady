pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'hxjerry/teedy'
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        DOCKER_CREDENTIALS_ID = 'dockerhub'
        DOCKER_SERVICE = 'teedy'
        DOCKER_PORTS = '8081 8082 8083'
        K8S_NAMESPACE = 'default'
        K8S_DEPLOYMENT = 'teedy'
        K8S_REPLICAS = '3'
        K8S_SERVICE_PORT = '8090'
        K8S_CONTAINER_PORT = '8080'
        K8S_IMAGE_PULL_SECRET = 'dockerhub'
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
                            docker service rm "$DOCKER_SERVICE"
                        fi

                        index=1
                        for port in $DOCKER_PORTS; do
                            service="$DOCKER_SERVICE-$index"

                            if docker service inspect "$service" >/dev/null 2>&1; then
                                docker service update \
                                    --with-registry-auth \
                                    --image "$DOCKER_IMAGE:$DOCKER_TAG" \
                                    --publish-rm 8080 \
                                    --publish-add "published=$port,target=8080" \
                                    "$service"
                            else
                                docker service create \
                                    --with-registry-auth \
                                    --name "$service" \
                                    --replicas 1 \
                                    --publish "published=$port,target=8080" \
                                    "$DOCKER_IMAGE:$DOCKER_TAG"
                            fi

                            index=$((index + 1))
                        done
                    '''
                }
            }
        }

        stage('Deploy Kubernetes') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh '''
                        set -eu
                        printf '%s' "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

                        kubectl create secret generic "$K8S_IMAGE_PULL_SECRET" \
                            --namespace "$K8S_NAMESPACE" \
                            --from-file=.dockerconfigjson="$HOME/.docker/config.json" \
                            --type=kubernetes.io/dockerconfigjson \
                            --dry-run=client \
                            -o yaml | kubectl apply -f -

                        cat <<EOF | kubectl apply --namespace "$K8S_NAMESPACE" -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $K8S_DEPLOYMENT
  labels:
    app: $K8S_DEPLOYMENT
spec:
  replicas: $K8S_REPLICAS
  selector:
    matchLabels:
      app: $K8S_DEPLOYMENT
  template:
    metadata:
      labels:
        app: $K8S_DEPLOYMENT
    spec:
      imagePullSecrets:
        - name: $K8S_IMAGE_PULL_SECRET
      containers:
        - name: $K8S_DEPLOYMENT
          image: $DOCKER_IMAGE:$DOCKER_TAG
          imagePullPolicy: Always
          ports:
            - containerPort: $K8S_CONTAINER_PORT
---
apiVersion: v1
kind: Service
metadata:
  name: $K8S_DEPLOYMENT
spec:
  type: NodePort
  selector:
    app: $K8S_DEPLOYMENT
  ports:
    - name: http
      port: $K8S_SERVICE_PORT
      targetPort: $K8S_CONTAINER_PORT
EOF

                        kubectl rollout status deployment/"$K8S_DEPLOYMENT" \
                            --namespace "$K8S_NAMESPACE" \
                            --timeout=180s
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
