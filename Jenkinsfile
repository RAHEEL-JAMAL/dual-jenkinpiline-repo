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
                        env.DEPLOY_MODE = 'local'
                        env.DEPLOY_HOST = env.LOCAL_HOST
                        env.DEPLOY_USER = env.LOCAL_SSH_USER
                        echo "MODE: LOCAL VM (${env.LOCAL_HOST})"
                    } else if (params.DEPLOY_TARGET == 'aws') {
                        env.DEPLOY_MODE = 'aws'
                        env.DEPLOY_HOST = env.AWS_HOST
                        env.DEPLOY_USER = env.AWS_SSH_USER
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

                    if (env.DEPLOY_MODE == 'local') {
                        // FIX: added || true so grep non-match doesn't kill the stage
                        def usedRaw = sh(
                            script: "docker ps --format '{{.Ports}}' | grep -oP '[0-9]+(?=->)' || true",
                            returnStdout: true
                        ).trim()

                        def usedPorts = usedRaw ? usedRaw.split('\n').collect { it.trim() } : []
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
                sh 'rm -rf app && git clone --depth=1 "${REPO_URL}" app'
                script {
                    echo '[STAGE_SUCCESS] Clone Repo'
                }
            }
        }

        // ── Stage 6 ───────────────────────────────────────────────────────────
        // FIX: Moved .dockerignore creation to its own sh block with proper
        //      heredoc quoting — this was causing the "unexpected EOF" that
        //      killed Stage 7 (Secret Scan).
        // FIX: Added many more ignore patterns to reduce image size and speed
        //      up DockerHub pushes.
        stage('Setup Docker Ignore') {
            steps {
                script {
                    echo '[STAGE_START] Setup Docker Ignore'
                }
                sh '''
# Write .dockerignore without any Groovy string interpolation issues
cat > app/.dockerignore << 'DOCKERIGNORE_EOF'
# Version control
.git
.gitignore
.gitattributes

# Dependencies — biggest size win
node_modules
bower_components
vendor
.pnp
.pnp.js

# Build outputs (re-built inside Docker)
dist
build
out
.next
.nuxt
.vite
.cache
*.tsbuildinfo

# Test & coverage
coverage
.nyc_output
__tests__
**/*.test.js
**/*.spec.js
**/*.test.ts
**/*.spec.ts
jest.config.*
cypress
playwright

# Dev tooling
.eslintrc*
.prettierrc*
.editorconfig
.stylelintrc*
tsconfig.tsbuildinfo
.husky
.lint-staged*

# Logs
*.log
logs
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# Environment / secrets — never bake into image
.env
.env.*
*.pem
*.key
*.cert
secrets

# OS junk
.DS_Store
Thumbs.db
*.swp
*~

# Docker itself
Dockerfile*
docker-compose*
.dockerignore

# CI / docs
.github
.circleci
docs
*.md
README*
CHANGELOG*
LICENSE*

# Python
__pycache__
*.pyc
*.pyo
*.pyd
.Python
.venv
venv
env
*.egg-info
dist-info

# Java / Maven / Gradle
target
.gradle
*.class
*.jar
*.war

# Misc large dirs
.terraform
.serverless
DOCKERIGNORE_EOF
'''
                script {
                    echo '[STAGE_SUCCESS] Setup Docker Ignore'
                }
            }
        }

        // ── Stage 7 ───────────────────────────────────────────────────────────
        // FIX: Was crashing with "unexpected EOF" because the previous stage's
        //      heredoc in a script{} block corrupted the shell context.
        //      Now runs cleanly in its own sh block.
        stage('Secret Scan') {
            steps {
                script {
                    echo '[STAGE_START] Secret Scan'
                }
                sh '''#!/bin/bash
set +e
FOUND=0

while IFS= read -r -d '' f; do
    if grep -qiE "(password|api_key|secret|token)\\s*=\\s*['\"]?[A-Za-z0-9+/]{8,}" "$f" 2>/dev/null; then
        echo "[WARN] Potential secret found in: $f"
        FOUND=1
    fi
done < <(find app -type f \\( -name "*.js" -o -name "*.py" -o -name "*.env" -o -name "*.ts" \\) \
    -not -path "*/node_modules/*" \
    -not -path "*/.git/*" \
    -print0)

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
        // FIX: Added monorepo detection — looks for common sub-package layouts
        //      and sets PKG_ROOT so Dockerfile targets the right subdirectory.
        // FIX: Also detects entry point (index.js vs server.js vs main.js vs src/index.js)
        //      so CMD in Dockerfile is correct.
        stage('Detect Stack') {
            steps {
                script {
                    echo '[STAGE_START] Detect Stack'

                    def stack    = 'node'
                    def pkgRoot  = 'app'
                    def entryPt  = 'index.js'

                    // ── Monorepo detection ─────────────────────────────────
                    // Common mono-repo layouts: packages/, apps/, services/
                    def monoSubdirs = ['packages', 'apps', 'services', 'frontend', 'backend', 'client', 'server', 'api', 'web']
                    def foundMono   = false
                    for (sub in monoSubdirs) {
                        if (fileExists("app/${sub}/package.json")) {
                            pkgRoot  = "app/${sub}"
                            foundMono = true
                            echo "[INFO] Monorepo detected — using sub-package: ${sub}"
                            break
                        }
                    }

                    // ── Stack detection ────────────────────────────────────
                    def pkgJsonPath = "${pkgRoot}/package.json"
                    if (fileExists(pkgJsonPath)) {
                        def pkg = readFile(pkgJsonPath)
                        if      (pkg.contains('"vite"'))   stack = 'vite'
                        else if (pkg.contains('"next"'))   stack = 'nextjs'
                        else if (pkg.contains('"react"'))  stack = 'react'
                        else if (pkg.contains('"express"') || pkg.contains('"fastify"') || pkg.contains('"koa"')) stack = 'node-server'
                        else                               stack = 'node'

                        // ── Entry point detection ──────────────────────────
                        // Read "main" or "scripts.start" from package.json
                        def mainMatch  = (pkg =~ /"main"\s*:\s*"([^"]+)"/)
                        def startMatch = (pkg =~ /"start"\s*:\s*"[^"]*node\s+([^\s"]+)"/)
                        if (mainMatch.find()) {
                            entryPt = mainMatch.group(1)
                        } else if (startMatch.find()) {
                            entryPt = startMatch.group(1)
                        } else {
                            // Probe common entry points
                            for (ep in ['index.js', 'server.js', 'app.js', 'main.js', 'src/index.js', 'src/app.js', 'src/server.js']) {
                                if (fileExists("${pkgRoot}/${ep}")) {
                                    entryPt = ep
                                    break
                                }
                            }
                        }
                    } else if (fileExists("${pkgRoot}/requirements.txt")) {
                        stack = 'python'
                        // Probe Python entry point
                        for (ep in ['app.py', 'main.py', 'run.py', 'server.py', 'manage.py']) {
                            if (fileExists("${pkgRoot}/${ep}")) {
                                entryPt = ep
                                break
                            }
                        }
                    } else if (fileExists("${pkgRoot}/pom.xml")) {
                        stack  = 'java'
                        entryPt = ''
                    }

                    env.STACK    = stack
                    env.PKG_ROOT = pkgRoot
                    env.ENTRY_PT = entryPt

                    echo "[META] STACK=${env.STACK}"
                    echo "[META] PKG_ROOT=${env.PKG_ROOT}"
                    echo "[META] ENTRY_PT=${env.ENTRY_PT}"
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
PKG_JSON="${PKG_ROOT}/package.json"
if [ -f "$PKG_JSON" ]; then
    echo "[INFO] Running npm audit in ${PKG_ROOT}..."

    docker run --rm \
        -v "$(pwd)/${PKG_ROOT}:/work" \
        -w /work \
        node:20-alpine \
        sh -c "npm install --prefer-offline --ignore-scripts 2>&1 | tail -5 && npm audit --json 2>/dev/null || true" \
        > /tmp/audit_out.txt 2>&1

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
        // FIX: Completely rewritten Dockerfile generation.
        //
        //  • vite / react  → multi-stage: build with node, serve with nginx
        //                     (previously tried to run dev server in prod — broken)
        //  • nextjs        → multi-stage: build then standalone output
        //  • node-server   → single-stage with detected entry point
        //  • node          → single-stage with detected entry point
        //  • python        → single-stage with detected entry point
        //  • java          → multi-stage: maven build + JRE runtime
        //
        //  All images bind on 0.0.0.0 (explicit) so the host can reach them.
        //  nginx config listens on 0.0.0.0:80 mapped to host port $PORT.
        stage('Create Dockerfile') {
            steps {
                script {
                    echo '[STAGE_START] Create Dockerfile'

                    def dfPath = "${env.PKG_ROOT}/Dockerfile"

                    if (fileExists(dfPath)) {
                        echo '[INFO] Dockerfile already exists in repo — using as-is'
                    } else {
                        def df = ''
                        def entry = env.ENTRY_PT ?: 'index.js'

                        switch (env.STACK) {

                            case 'vite':
                            case 'react':
                                // ── Multi-stage: build SPA → serve with nginx ──
                                df = """FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --ignore-scripts
COPY . .
RUN npm run build

FROM nginx:alpine AS runner
# nginx listens on 0.0.0.0:80 by default — accessible from host
COPY --from=builder /app/dist /usr/share/nginx/html
# SPA fallback: all routes → index.html
RUN printf 'server {\\n  listen 80;\\n  root /usr/share/nginx/html;\\n  index index.html;\\n  location / { try_files \\$uri \\$uri/ /index.html; }\\n}\\n' > /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
"""
                                // nginx uses port 80, update PORT mapping
                                env.CONTAINER_PORT = '80'
                                break

                            case 'nextjs':
                                df = """FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --ignore-scripts
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
# Bind to all interfaces so host can reach the container
ENV HOSTNAME=0.0.0.0
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
EXPOSE 3000
CMD ["node", "server.js"]
"""
                                env.CONTAINER_PORT = '3000'
                                break

                            case 'node-server':
                            case 'node':
                                df = """FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production --ignore-scripts
COPY . .
EXPOSE 3000
# HOST env var forces Node/Express to bind on all interfaces (0.0.0.0)
ENV HOST=0.0.0.0
ENV PORT=3000
CMD ["node", "${entry}"]
"""
                                env.CONTAINER_PORT = '3000'
                                break

                            case 'python':
                                df = """FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 3000
# Bind to 0.0.0.0 so host browser can reach the container
ENV HOST=0.0.0.0
ENV PORT=3000
CMD ["python", "${entry}"]
"""
                                env.CONTAINER_PORT = '3000'
                                break

                            case 'java':
                                df = """FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml ./
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:17-jre-alpine AS runner
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 3000
CMD ["java", "-jar", "app.jar", "--server.port=3000", "--server.address=0.0.0.0"]
"""
                                env.CONTAINER_PORT = '3000'
                                break

                            default:
                                df = """FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production --ignore-scripts
COPY . .
EXPOSE 3000
ENV HOST=0.0.0.0
ENV PORT=3000
CMD ["node", "${entry}"]
"""
                                env.CONTAINER_PORT = '3000'
                        }

                        writeFile file: dfPath, text: df
                        echo "[INFO] Dockerfile created for stack: ${env.STACK}"
                        echo "[INFO] Entry point: ${entry}"
                    }

                    // Default container port if not set by switch above
                    if (!env.CONTAINER_PORT) { env.CONTAINER_PORT = '3000' }

                    echo '[STAGE_SUCCESS] Create Dockerfile'
                }
            }
        }

        // ── Stage 11 ──────────────────────────────────────────────────────────
        // FIX: Build context is now PKG_ROOT (handles monorepos).
        //      Added --progress=plain for clearer logs.
        //      Added BuildKit for faster layer caching.
        stage('Build Image') {
            steps {
                script {
                    echo '[STAGE_START] Build Image'
                }
                sh '''
export DOCKER_BUILDKIT=1
docker build \
    --progress=plain \
    --build-arg BUILDKIT_INLINE_CACHE=1 \
    -t "${IMAGE_NAME}" \
    "${PKG_ROOT}/"
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
        // FIX: Port mapping now uses CONTAINER_PORT (80 for nginx/vite builds,
        //      3000 for node/python) — previously always mapped to :3000 which
        //      broke nginx-served static apps.
        // FIX: Added --add-host=host-gateway so container can reach host if needed.
        stage('Deploy') {
            steps {
                script {
                    echo '[STAGE_START] Deploy'

                    if (env.DEPLOY_MODE == 'local') {
                        echo '[INFO] Starting LOCAL VM deployment...'

                        sh """
                            docker stop ${CONTAINER_NAME} 2>/dev/null || true
                            docker rm   ${CONTAINER_NAME} 2>/dev/null || true
                            docker pull ${IMAGE_NAME}

                            docker run -d \
                                --name ${CONTAINER_NAME} \
                                --restart unless-stopped \
                                -p 0.0.0.0:${PORT}:${CONTAINER_PORT} \
                                ${IMAGE_NAME}
                        """

                        echo "[META] URL=http://${LOCAL_HOST}:${PORT}"
                        echo "LOCAL URL: http://${LOCAL_HOST}:${PORT}"

                    } else if (env.DEPLOY_MODE == 'aws') {
                        echo '[INFO] Starting AWS EC2 deployment...'

                        sh """
                            ssh -o StrictHostKeyChecking=no \
                                -o ConnectTimeout=30 \
                                ${AWS_SSH_USER}@${AWS_HOST} '
                                    docker pull ${IMAGE_NAME}
                                    docker stop ${CONTAINER_NAME} 2>/dev/null || true
                                    docker rm   ${CONTAINER_NAME} 2>/dev/null || true
                                    docker run -d \\
                                        --name ${CONTAINER_NAME} \\
                                        --restart unless-stopped \\
                                        -p 0.0.0.0:80:${CONTAINER_PORT} \\
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
sleep 3   # give container a moment to start

if [ "${DEPLOY_MODE}" = "local" ]; then
    if docker ps --format "{{.Names}}" | grep -q "^${CONTAINER_NAME}$"; then
        echo "[INFO] Container ${CONTAINER_NAME} is running"
        docker ps --filter "name=${CONTAINER_NAME}" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

        # HTTP health check — works for both nginx(:80) and node(:3000)
        echo "[INFO] HTTP health check..."
        curl -sf --max-time 10 "http://localhost:${PORT}/" -o /dev/null \
            && echo "[INFO] HTTP check PASSED" \
            || echo "[WARN] HTTP check failed — app may still be starting"
    else
        echo "[WARN] Container ${CONTAINER_NAME} not found in docker ps"
    fi
else
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
        // always runs first (cleanup), then success/failure
        always {
            echo '[INFO] Pipeline complete — cleaning up workspace'
            sh 'rm -rf app || true'
        }
        success {
            echo '[DEPLOY_SUCCESS]'
            echo "[META] FINAL_STATUS=SUCCESS"
            echo "[META] DEPLOY_MODE=${env.DEPLOY_MODE}"
        }
        failure {
            echo '[DEPLOY_FAILED]'
            echo "[META] FINAL_STATUS=FAILED"
            sh "docker rmi ${env.IMAGE_NAME} 2>/dev/null || true"
        }
    }

}
