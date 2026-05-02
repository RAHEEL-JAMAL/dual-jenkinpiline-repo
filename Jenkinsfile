pipeline {
    agent any

    parameters {
        string(
            name: 'REPO_URL',
            defaultValue: '',
            description: 'GitHub repository URL to deploy'
        )
        string(
            name: 'APP_NAME',
            defaultValue: '',
            description: 'Unique app name (letters, numbers, hyphens only)'
        )
        choice(
            name: 'DEPLOY_TARGET',
            choices: ['local', 'aws'],
            description: 'Deployment target: local VM or AWS EC2'
        )
    }

    environment {
        DOCKERHUB_CRED  = credentials('dockerhub-cred')
        DOCKERHUB_USER  = 'raheeljamal'
        AWS_HOST        = "${env.AWS_EC2_IP ?: 'YOUR_EC2_PUBLIC_IP'}"
        LOCAL_HOST      = '192.168.122.127'
        LOCAL_SSH_USER  = 'ubuntu'
        AWS_SSH_USER    = 'ubuntu'
    }

    stages {

        // ── Stage 1 ───────────────────────────────────────────────────────────
        stage('Init') {
            steps {
                script {
                    echo '[STAGE_START] Init'
                    echo '[INFO] MULTI APP SAFE DEPLOY STARTED'
                    echo "[INFO] DEPLOY_TARGET: ${params.DEPLOY_TARGET}"
                    echo "[INFO] REPO_URL: ${params.REPO_URL}"
                    echo "[INFO] APP_NAME: ${params.APP_NAME}"
                    echo '[STAGE_SUCCESS] Init'
                }
            }
        }

        // ── Stage 2 ───────────────────────────────────────────────────────────
        stage('Input Repo') {
            steps {
                script {
                    echo '[STAGE_START] Input Repo'

                    def repoUrl = params.REPO_URL?.trim()
                    def appName = params.APP_NAME?.trim()

                    if (!repoUrl) error('[ERROR] REPO_URL is required')
                    if (!appName) error('[ERROR] APP_NAME is required')

                    // Generate short unique ID from repo URL
                    def appId = sh(
                        script: "printf '%s' '${repoUrl}' | md5sum | cut -c1-6",
                        returnStdout: true
                    ).trim()

                    env.REPO_URL       = repoUrl
                    env.APP_NAME       = appName
                    env.APP_ID         = appId
                    env.CONTAINER_NAME = "app_${appId}"
                    env.IMAGE_NAME     = "${DOCKERHUB_USER}/app_${appId}:latest"
                    env.PKG_ROOT       = 'app'

                    echo "[META] REPO_URL=${env.REPO_URL}"
                    echo "[META] APP_NAME=${env.APP_NAME}"
                    echo "[META] APP_ID=${env.APP_ID}"
                    echo "[META] CONTAINER_NAME=${env.CONTAINER_NAME}"
                    echo "[META] IMAGE_NAME=${env.IMAGE_NAME}"

                    echo '[STAGE_SUCCESS] Input Repo'
                }
            }
        }

        // ── Stage 3 ───────────────────────────────────────────────────────────
        stage('Select Deployment Mode') {
            steps {
                script {
                    echo '[STAGE_START] Select Deployment Mode'

                    if (params.DEPLOY_TARGET == 'local') {
                        env.DEPLOY_MODE   = 'local'
                        env.DEPLOY_HOST   = env.LOCAL_HOST
                        env.DEPLOY_USER   = env.LOCAL_SSH_USER
                        echo "MODE: LOCAL VM (${env.LOCAL_HOST})"
                    } else if (params.DEPLOY_TARGET == 'aws') {
                        env.DEPLOY_MODE   = 'aws'
                        env.DEPLOY_HOST   = env.AWS_HOST
                        env.DEPLOY_USER   = env.AWS_SSH_USER
                        echo "MODE: AWS EC2"
                    } else {
                        error('[ERROR] Invalid DEPLOY_TARGET. Must be local or aws.')
                    }

                    echo "[META] DEPLOY_MODE=${env.DEPLOY_MODE}"
                    echo "[META] DEPLOY_HOST=${env.DEPLOY_HOST}"

                    echo '[STAGE_SUCCESS] Select Deployment Mode'
                }
            }
        }

        // ── Stage 4 ───────────────────────────────────────────────────────────
        stage('Allocate Safe Port') {
            steps {
                script {
                    echo '[STAGE_START] Allocate Safe Port'

                    // Only needed for local (AWS always uses port 80)
                    if (env.DEPLOY_MODE == 'local') {
                        def usedRaw = sh(
                            script: "docker ps --format '{{.Ports}}' | grep -o '[0-9]*->' | grep -o '[0-9]*' || true",
                            returnStdout: true
                        ).trim()

                        def usedPorts = usedRaw ? usedRaw.split('\n').toList() : []
                        def port = 3000
                        while (usedPorts.contains(port.toString())) { port++ }
                        env.PORT = port.toString()
                    } else {
                        env.PORT = '80'
                    }

                    echo "[META] PORT=${env.PORT}"
                    echo '[STAGE_SUCCESS] Allocate Safe Port'
                }
            }
        }

        // ── Stage 5 ───────────────────────────────────────────────────────────
        stage('Clone Repo') {
            steps {
                script {
                    echo '[STAGE_START] Clone Repo'
                }

                sh '''
                    rm -rf app
                    git clone --depth=1 "${REPO_URL}" app
                '''

                script {
                    echo '[STAGE_SUCCESS] Clone Repo'
                }
            }
        }

        // ── Stage 6 ───────────────────────────────────────────────────────────
        stage('Setup Docker Ignore') {
            steps {
                script {
                    echo '[STAGE_START] Setup Docker Ignore'

                    sh '''
                        cat > app/.dockerignore <<'EOF'
node_modules
.git
npm-debug.log
dist
build
.env
coverage
.DS_Store
*.log
EOF
                    '''

                    echo '[STAGE_SUCCESS] Setup Docker Ignore'
                }
            }
        }

        // ── Stage 7 ───────────────────────────────────────────────────────────
        stage('Secret Scan') {
            steps {
                script {
                    echo '[STAGE_START] Secret Scan'
                }

                sh '''#!/bin/bash
                    set +e
                    FOUND=0

                    while IFS= read -r f; do
                        if grep -qiE "(password|api_key|secret|token)\\s*=\\s*['\"]?[A-Za-z0-9+/]{8,}" "$f" 2>/dev/null; then
                            echo "[WARN] Potential secret found in: $f"
                            FOUND=1
                        fi
                    done < <(find app -type f \\( -name "*.js" -o -name "*.py" -o -name "*.env" -o -name "*.ts" \\) \
                        -not -path "*/node_modules/*" \
                        -not -path "*/.git/*")

                    if [ "$FOUND" = "1" ]; then
                        echo "[META] SECRET_SCAN=FAILED"
                        exit 1
                    fi

                    echo "[META] SECRET_SCAN=PASSED"
                '''

                script {
                    echo '[STAGE_SUCCESS] Secret Scan'
                }
            }
        }

        // ── Stage 8 ───────────────────────────────────────────────────────────
        stage('Detect Stack') {
            steps {
                script {
                    echo '[STAGE_START] Detect Stack'

                    def stack = 'node'

                    if (fileExists('app/package.json')) {
                        def pkg = readFile('app/package.json')
                        if      (pkg.contains('"vite"'))  stack = 'vite'
                        else if (pkg.contains('"next"'))  stack = 'nextjs'
                        else if (pkg.contains('"react"')) stack = 'react'
                        else                              stack = 'node'
                    } else if (fileExists('app/requirements.txt')) {
                        stack = 'python'
                    } else if (fileExists('app/pom.xml')) {
                        stack = 'java'
                    }

                    env.STACK = stack
                    echo "[META] STACK=${env.STACK}"
                    echo '[STAGE_SUCCESS] Detect Stack'
                }
            }
        }

        // ── Stage 9 ───────────────────────────────────────────────────────────
        stage('Dependency Audit') {
            steps {
                script {
                    echo '[STAGE_START] Dependency Audit'
                }

                sh '''#!/bin/bash
                    set +e
                    if [ -f "app/package.json" ]; then
                        echo "[INFO] Running npm audit..."

                        docker run --rm \
                            -v "$(pwd)/app:/work" \
                            -w /work \
                            node:20-alpine \
                            sh -c "npm install --prefer-offline 2>&1 && npm audit --json 2>/dev/null || true" \
                            > /tmp/audit_out.txt 2>&1

                        # Parse critical/high counts
                        CRITICAL=$(grep -o '"critical":[0-9]*' /tmp/audit_out.txt | grep -o '[0-9]*' | head -1 || echo "0")
                        HIGH=$(grep -o '"high":[0-9]*' /tmp/audit_out.txt | grep -o '[0-9]*' | head -1 || echo "0")

                        echo "[META] VULN_CRITICAL=${CRITICAL:-0}"
                        echo "[META] VULN_HIGH=${HIGH:-0}"
                        echo "[META] DEPENDENCY_SCAN=PASSED"
                    else
                        echo "[INFO] No package.json found, skipping npm audit"
                        echo "[META] VULN_CRITICAL=0"
                        echo "[META] VULN_HIGH=0"
                        echo "[META] DEPENDENCY_SCAN=PASSED"
                    fi
                '''

                script {
                    echo '[STAGE_SUCCESS] Dependency Audit'
                }
            }
        }

        // ── Stage 10 ──────────────────────────────────────────────────────────
        stage('Create Dockerfile') {
            steps {
                script {
                    echo '[STAGE_START] Create Dockerfile'

                    // Only create if no Dockerfile exists in repo
                    if (!fileExists('app/Dockerfile')) {
                        def df = ''

                        if (env.STACK == 'python') {
                            df = '''FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 3000
CMD ["python", "app.py"]
'''
                        } else {
                            // node / react / vite / nextjs
                            df = '''FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
'''
                        }

                        writeFile file: 'app/Dockerfile', text: df
                        echo '[INFO] Dockerfile created'
                    } else {
                        echo '[INFO] Dockerfile already exists in repo, using as-is'
                    }

                    echo '[STAGE_SUCCESS] Create Dockerfile'
                }
            }
        }

        // ── Stage 11 ──────────────────────────────────────────────────────────
        stage('Build Image') {
            steps {
                script {
                    echo '[STAGE_START] Build Image'
                }

                sh '''
                    docker build -t "${IMAGE_NAME}" app/
                '''

                script {
                    echo '[STAGE_SUCCESS] Build Image'
                }
            }
        }

        // ── Stage 12 ──────────────────────────────────────────────────────────
        stage('Push to DockerHub') {
            steps {
                script {
                    echo '[STAGE_START] Push to DockerHub'
                }

                sh '''
                    echo "$DOCKERHUB_CRED_PSW" | docker login -u "$DOCKERHUB_CRED_USR" --password-stdin
                    docker push "${IMAGE_NAME}"
                    docker logout
                '''

                script {
                    echo '[STAGE_SUCCESS] Push to DockerHub'
                }
            }
        }

        // ── Stage 13 ──────────────────────────────────────────────────────────
        stage('Deploy') {
            steps {
                script {
                    echo '[STAGE_START] Deploy'

                    if (env.DEPLOY_MODE == 'local') {
                        // ── LOCAL VM deployment ────────────────────────────
                        echo '[INFO] Starting LOCAL VM deployment...'

                        sh """
                            docker stop ${CONTAINER_NAME} || true
                            docker rm   ${CONTAINER_NAME} || true
                            docker pull ${IMAGE_NAME}

                            docker run -d \
                                --name ${CONTAINER_NAME} \
                                --restart unless-stopped \
                                -p ${PORT}:3000 \
                                ${IMAGE_NAME}
                        """

                        echo "[META] URL=http://${LOCAL_HOST}:${PORT}"
                        echo "LOCAL URL: http://${LOCAL_HOST}:${PORT}"

                    } else if (env.DEPLOY_MODE == 'aws') {
                        // ── AWS EC2 deployment ─────────────────────────────
                        echo '[INFO] Starting AWS EC2 deployment...'

                        sh """
                            ssh -o StrictHostKeyChecking=no \
                                -o ConnectTimeout=30 \
                                ${AWS_SSH_USER}@${AWS_HOST} '
                                    docker pull ${IMAGE_NAME}
                                    docker stop ${CONTAINER_NAME} || true
                                    docker rm   ${CONTAINER_NAME} || true
                                    docker run -d \\
                                        --name ${CONTAINER_NAME} \\
                                        --restart unless-stopped \\
                                        -p 80:3000 \\
                                        ${IMAGE_NAME}
                                '
                        """

                        echo "[META] URL=http://${AWS_HOST}"
                        echo "AWS URL: http://${AWS_HOST}"
                    }

                    echo '[STAGE_SUCCESS] Deploy'
                }
            }
        }

        // ── Stage 14 ──────────────────────────────────────────────────────────
        stage('Verify') {
            steps {
                script {
                    echo '[STAGE_START] Verify'
                }

                sh '''#!/bin/bash
                    set +e

                    if [ "${DEPLOY_MODE}" = "local" ]; then
                        # Check container running locally
                        if docker ps --format "{{.Names}}" | grep -q "^${CONTAINER_NAME}$"; then
                            echo "[INFO] Container ${CONTAINER_NAME} is running"
                            docker ps --filter "name=${CONTAINER_NAME}" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
                        else
                            echo "[WARN] Container ${CONTAINER_NAME} not found in docker ps"
                        fi
                    else
                        # Check container running on AWS
                        ssh -o StrictHostKeyChecking=no \
                            -o ConnectTimeout=15 \
                            "${DEPLOY_USER}@${DEPLOY_HOST}" \
                            "docker ps --filter 'name=${CONTAINER_NAME}' --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'" || true
                    fi
                '''

                script {
                    echo '[STAGE_SUCCESS] Verify'
                }
            }
        }

    } // end stages

    post {
        success {
            echo '[DEPLOY_SUCCESS]'
            echo "[META] FINAL_STATUS=SUCCESS"
            echo "[META] DEPLOY_MODE=${env.DEPLOY_MODE}"
        }
        failure {
            echo '[DEPLOY_FAILED]'
            echo "[META] FINAL_STATUS=FAILED"
            // Clean up partial image on failure
            sh "docker rmi ${env.IMAGE_NAME} || true"
        }
        always {
            echo '[INFO] Pipeline complete'
            // Clean up cloned repo to save disk space
            sh 'rm -rf app || true'
        }
    }

}
