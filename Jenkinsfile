pipeline {
    agent
    {
        label 'docker'
    }
    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        REGISTRY_URL = "192.168.56.12:5000"
    }
    stages {
        stage('Test the API') {
            when { 
                changeset "services/api/**" 
            }
            agent {
                docker {
                    image 'python:3.12-slim'
                    // This ensures the container runs as the same user as Jenkins 
                    // to avoid permission issues with the workspace
                    args '-u root' 
                }
            }
            steps {
                sh '''
                # We are now INSIDE the python container
                # The workspace is automatically mounted at /home/jenkins/agent/workspace/...
                cd services/api
                pip install flask redis pytest
                python3 -m pytest tests/
                '''
            }
        }
        stage('Test the Archiver') {
            when {
                changeset "services/archiver/**"
            }
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-21'
                }
            }
            steps {
                sh 'cd services/archiver && mvn -B test'
            }
        }
        stage('Test the Frontend') {
            when {
                changeset "services/frontend/**"
            }
            steps {
                sh '''
                set -eu
                image="task-manager-frontend-test:${BUILD_NUMBER}"
                container="task-manager-frontend-test-${BUILD_NUMBER}-${EXECUTOR_NUMBER}"
                docker build -t "$image" services/frontend
                docker run -d --name "$container" --add-host api:127.0.0.1 --add-host monitor:127.0.0.1 "$image"
                trap 'docker rm -f "$container" >/dev/null 2>&1 || true; docker image rm "$image" >/dev/null 2>&1 || true' EXIT
                docker run --rm --network "container:$container" \\
                    -v "$WORKSPACE/services/frontend/test:/tests:ro" \\
                    alpine:3.20 sh -c 'apk add --no-cache curl && sh /tests/smoke_test.sh'
                '''
            }
        }
        stage('Test the Monitor') {
            when {
                changeset "services/monitor/**"
            }
            steps {
                sh '''
                set -eu
                network="task-manager-monitor-test-${BUILD_NUMBER}-${EXECUTOR_NUMBER}"
                redis="task-manager-monitor-redis-${BUILD_NUMBER}-${EXECUTOR_NUMBER}"
                docker network create "$network"
                trap 'docker rm -f "$redis" >/dev/null 2>&1 || true; docker network rm "$network" >/dev/null 2>&1 || true' EXIT
                docker run -d --name "$redis" --network "$network" --network-alias redis redis:7-alpine
                docker run --rm --network "$network" -e REDIS_HOST=redis \\
                    -v "$WORKSPACE/services/monitor:/src" -w /src golang:1.21 \\
                    sh -c 'go mod tidy && go get github.com/stretchr/testify/assert && go test -v ./...'
                '''
            }
        }
        stage('Build the API') {
            when { 
                changeset "services/api/**" 
            }
            steps {
                sh '''
				cd services/api
				docker build -t ${REGISTRY_URL}/task-manager-api:${IMAGE_TAG} .
				docker push ${REGISTRY_URL}/task-manager-api:${IMAGE_TAG}
				'''
            }
        }
        stage('Build the Archiver') {
            when { 
                changeset "services/archiver/**" 
            }
            steps {
                sh '''
				cd services/archiver
				docker build -t ${REGISTRY_URL}/task-manager-archiver:${IMAGE_TAG} .
				docker push ${REGISTRY_URL}/task-manager-archiver:${IMAGE_TAG}
				'''
            }
        }
        stage('Build the Frontend') {
            when { 
                changeset "services/frontend/**" 
            }
            steps {
                sh '''
				cd services/frontend
				docker build -t ${REGISTRY_URL}/task-manager-frontend:${IMAGE_TAG} .
				docker push ${REGISTRY_URL}/task-manager-frontend:${IMAGE_TAG}
				'''
            }
        }
        stage('Build the Monitor') {
            when { 
                changeset "services/monitor/**" 
            }
            steps {
                sh '''
				cd services/monitor
				docker build -t ${REGISTRY_URL}/task-manager-monitor:${IMAGE_TAG} .
				docker push ${REGISTRY_URL}/task-manager-monitor:${IMAGE_TAG}
				'''
            }
        }
        stage('Clean')
        {
            steps
            {
                cleanWs()
            }
        }
    }
}
