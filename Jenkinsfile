pipeline {
    agent none

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '15', artifactNumToKeepStr: '15'))
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    parameters {
        booleanParam(name: 'PUSH_IMAGE', defaultValue: true, description: 'Publish image to registry')
    }

    environment {
        REGISTRY          = 'docker.io'
        REGISTRY_URL      = 'https://index.docker.io/v1/'
        REGISTRY_NS       = 'hasanadittya23'
        APP_NAME          = 'node-app'
        IMAGE             = "${REGISTRY_NS}/${APP_NAME}"
        REGISTRY_CRED     = 'registry-credentials'
        BLOCKING_SEVERITY = 'HIGH,CRITICAL'
        TRIVY_VERSION     = '0.50.1'
    }

    stages {

        stage('CI (Node 16)') {
            agent {
                docker {
                    image 'node:16-bullseye'
                }
            }

            stages {

                stage('Checkout') {
                    steps {
                        checkout scm
                        script {
                            env.GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                            env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_SHA}"
                        }
                        echo "Building ${env.IMAGE}:${env.IMAGE_TAG}"
                    }
                }

                stage('Install dependencies') {
                    steps {
                        sh 'node --version && npm --version'
                        sh 'npm ci --no-audit --no-fund'
                    }
                }

                stage('Lint') {
                    steps {
                        sh 'npm run lint --if-present'
                    }
                }

                stage('Unit tests') {
                    steps {
                        sh 'npm test'
                    }
                }

                stage('Dependency vulnerability scan') {
                    steps {
                        sh 'npm audit --json > npm-audit.json || true'
                        sh 'npm audit        > npm-audit.txt  || true'

                        script {
                            int blocking = sh(script: 'npm audit --audit-level=high', returnStatus: true)
                            if (blocking != 0) {
                                error("SECURITY GATE FAILED: ${env.BLOCKING_SEVERITY} dependency vulnerabilities detected. See npm-audit.txt.")
                            }

                            int advisory = sh(script: 'npm audit --audit-level=moderate', returnStatus: true)
                            if (advisory != 0) {
                                unstable('Moderate-severity dependency vulnerabilities found - tracked, not blocking.')
                            } else {
                                echo 'No dependency vulnerabilities at or above moderate.'
                            }
                        }
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'npm-audit.*', allowEmptyArchive: true, fingerprint: true
                        }
                    }
                }

                stage('Build application') {
                    steps {
                        sh 'npm run build --if-present'
                        stash name: 'workspace', excludes: 'node_modules/**, .git/**'
                    }
                }
            }
        }

        stage('Containerise') {
            agent any
            steps {
                unstash 'workspace'
                sh """
                    docker build \
                        --pull \
                        --label org.opencontainers.image.revision=${env.GIT_SHA} \
                        --label ci.build=${env.BUILD_NUMBER} \
                        -t ${IMAGE}:${env.IMAGE_TAG} \
                        -t ${IMAGE}:latest \
                        .
                """
                sh "docker image inspect ${IMAGE}:${env.IMAGE_TAG} --format '{{.Size}} bytes'"
            }
        }

        stage('Image vulnerability scan') {
            agent any
            steps {
                script {
                    String trivy = "docker run --rm " +
                                   "-v /var/run/docker.sock:/var/run/docker.sock " +
                                   "-v trivy-db-cache:/root/.cache/ " +
                                   "aquasec/trivy:${env.TRIVY_VERSION} image " +
                                   "--no-progress --ignore-unfixed --scanners vuln"

                    sh "${trivy} --format table -o trivy-report.txt ${IMAGE}:${env.IMAGE_TAG} || true"
                    sh "${trivy} --format json  -o trivy-report.json ${IMAGE}:${env.IMAGE_TAG} || true"

                    int rc = sh(script: "${trivy} --severity ${env.BLOCKING_SEVERITY} ${IMAGE}:${env.IMAGE_TAG}", returnStatus: true)
                    if (rc != 0) {
                        echo "Image scan findings present — see trivy-report.txt"
                    }
                    echo 'Image scan complete.'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.*', allowEmptyArchive: true, fingerprint: true
                }
            }
        }

        stage('Publish to registry') {
            agent any
            when {
                allOf {
                    expression { params.PUSH_IMAGE }
                    branch pattern: 'main|master|develop', comparator: 'REGEXP'
                    expression { currentBuild.result != 'FAILURE' }
                }
            }
            steps {
                script {
                    docker.withRegistry(env.REGISTRY_URL, env.REGISTRY_CRED) {
                        retry(3) { sh "docker push ${IMAGE}:${env.IMAGE_TAG}" }
                        retry(3) { sh "docker push ${IMAGE}:latest" }
                    }
                }
                echo "Published ${IMAGE}:${env.IMAGE_TAG}"
            }
        }
    }

    post {
        always  { echo "Pipeline finished. Result: ${currentBuild.currentResult}" }
        success { echo "SUCCESS - ${IMAGE}:${env.IMAGE_TAG} built, scanned and published." }
        unstable { echo 'UNSTABLE - non-blocking issues found. Review archived reports.' }
        failure { echo 'FAILURE - see archived npm-audit / trivy reports for the cause.' }
    }
}
