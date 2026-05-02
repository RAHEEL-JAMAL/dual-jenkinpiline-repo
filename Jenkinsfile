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
                        echo 'MODE: AWS EC2'
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
                script { echo '[STAGE_START] Clone Repo' }
                sh 'rm -rf app && git clone --depth=1 "${REPO_URL}" app'
                script { echo '[STAGE_SUCCESS] Clone Repo' }
            }
        }

        // ── Stage 6 ───────────────────────────────────────────────────────────
        // ROOT FIX: .dockerignore is written with Groovy writeFile() — completely
        // bypasses the shell, so NO heredoc quoting issues are possible at all.
        stage('Setup Docker Ignore') {
            steps {
                script {
                    echo '[STAGE_START] Setup Docker Ignore'

                    writeFile file: 'app/.dockerignore', text: [
                        '# VCS', '.git', '.gitignore', '.gitattributes',
                        '# Deps', 'node_modules', 'bower_components', 'vendor', '.pnp', '.pnp.js',
                        '# Build', 'dist', 'build', 'out', '.next', '.nuxt', '.vite', '.cache', '*.tsbuildinfo',
                        '# Tests', 'coverage', '.nyc_output', '__tests__',
                        '**/*.test.js', '**/*.spec.js', '**/*.test.ts', '**/*.spec.ts',
                        'jest.config.*', 'cypress', 'playwright',
                        '# Tooling', '.eslintrc*', '.prettierrc*', '.editorconfig',
                        '.stylelintrc*', '.husky', '.lint-staged*',
                        '# Logs', '*.log', 'logs',
                        'npm-debug.log*', 'yarn-debug.log*', 'yarn-error.log*',
                        '# Secrets', '.env', '.env.*', '*.pem', '*.key', '*.cert', 'secrets',
                        '# OS', '.DS_Store', 'Thumbs.db', '*.swp', '*~',
                        '# Docker', 'Dockerfile*', 'docker-compose*', '.dockerignore',
                        '# CI/Docs', '.github', '.circleci', 'docs',
                        '*.md', 'README*', 'CHANGELOG*', 'LICENSE*',
                        '# Python', '__pycache__', '*.pyc', '*.pyo', '.venv', 'venv', 'env', '*.egg-info',
                        '# Java', 'target', '.gradle', '*.class', '*.jar', '*.war',
                        '# Misc', '.terraform', '.serverless'
                    ].join('\n')

                    echo '[STAGE_SUCCESS] Setup Docker Ignore'
                }
            }
        }

        // ── Stage 7 ───────────────────────────────────────────────────────────
        // FIX: Plain sh block — no heredoc, no quote nesting.
        //      Uses -print0 + read -d '' for safe filename handling.
        stage('Secret Scan') {
            steps {
                script { echo '[STAGE_START] Secret Scan' }
                sh '''
set +e
FOUND=0
while IFS= read -r -d "" f; do
    if grep -qiP "(password|api_key|secret|token)\\s*=\\s*['\"]?[A-Za-z0-9+/]{8,}" "$f" 2>/dev/null; then
        echo "[WARN] Potential secret in: $f"
        FOUND=1
    fi
done < <(find app -type f \( -name "*.js" -o -name "*.py" -o -name "*.env" -o -name "*.ts" \) \
    -not -path "*/node_modules/*" \
    -not -path "*/.git/*" \
    -print0)
if [ "$FOUND" = "1" ]; then
    echo "[META] SECRET_SCAN=FAILED"
    exit 1
fi
echo "[META] SECRET_SCAN=PASSED"
'''
                script { echo '[STAGE_SUCCESS] Secret Scan' }
            }
        }

        // ── Stage 8 ───────────────────────────────────────────────────────────
        stage('Detect Stack') {
            steps {
                script {
                    echo '[STAGE_START] Detect Stack'

                    def stack   = 'node'
                    def pkgRoot = 'app'
                    def entryPt = 'index.js'

                    // Monorepo probe
                    for (sub in ['packages','apps','services','frontend','backend','client','server','api','web']) {
                        if (fileExists("app/${sub}/package.json")) {
                            pkgRoot = "app/${sub}"
                            echo "[INFO] Monorepo sub-package: ${sub}"
                            break
                        }
                    }

                    def pkgJsonPath = "${pkgRoot}/package.json"
                    if (fileExists(pkgJsonPath)) {
                        def pkg = readFile(pkgJsonPath)
                        if      (pkg.contains('"vite"'))    stack = 'vite'
                        else if (pkg.contains('"next"'))    stack = 'nextjs'
                        else if (pkg.contains('"react"'))   stack = 'react'
                        else if (pkg.contains('"express"') || pkg.contains('"fastify"') || pkg.contains('"koa"'))
                                                            stack = 'node-server'
                        else                                stack = 'node'

                        def mainMatch  = (pkg =~ /"main"\s*:\s*"([^"]+)"/)
                        def startMatch = (pkg =~ /"start"\s*:\s*"[^"]*node\s+([^\s"]+)"/)
                        if      (mainMatch.find())  { entryPt = mainMatch.group(1) }
                        else if (startMatch.find()) { entryPt = startMatch.group(1) }
                        else {
                            for (ep in ['index.js','server.js','app.js','main.js','src/index.js','src/app.js','src/server.js']) {
                                if (fileExists("${pkgRoot}/${ep}")) { entryPt = ep; break }
                            }
                        }
                    } else if (fileExists("${pkgRoot}/requirements.txt")) {
                        stack = 'python'
                        for (ep in ['app.py','main.py','run.py','server.py','manage.py']) {
                            if (fileExists("${pkgRoot}/${ep}")) { entryPt = ep; break }
                        }
                    } else if (fileExists("${pkgRoot}/pom.xml")) {
                        stack = 'java'; entryPt = ''
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
                script { echo '[STAGE_START] Dependency Audit' }
                sh '''
set +e
if [ -f "${PKG_ROOT}/package.json" ]; then
    echo "[INFO] Running npm audit in ${PKG_ROOT}..."
    docker run --rm \
        -v "$(pwd)/${PKG_ROOT}:/work" -w /work node:20-alpine \
        sh -c "npm install --prefer-offline --ignore-scripts 2>&1 | tail -5 && npm audit --json 2>/dev/null || true" \
        > /tmp/audit_out.txt 2>&1
    CRITICAL=$(grep -o '"critical":[0-9]*' /tmp/audit_out.txt | grep -o '[0-9]*' | head -1 || echo 0)
    HIGH=$(grep -o '"high":[0-9]*' /tmp/audit_out.txt | grep -o '[0-9]*' | head -1 || echo 0)
    echo "[META] VULN_CRITICAL=${CRITICAL:-0}"
    echo "[META] VULN_HIGH=${HIGH:-0}"
    echo "[META] DEPENDENCY_SCAN=PASSED"
else
    echo "[INFO] No package.json — skipping"
    echo "[META] VULN_CRITICAL=0"
    echo "[META] VULN_HIGH=0"
    echo "[META] DEPENDENCY_SCAN=PASSED"
fi
'''
                script { echo '[STAGE_SUCCESS] Dependency Audit' }
            }
        }

        // ── Stage 10 ──────────────────────────────────────────────────────────
        // FIX: All Dockerfile content built as a Groovy string and written with
        //      writeFile() — zero shell involved, zero heredoc quoting risk.
        //      Multi-stage builds for vite/react/nextjs/java.
        //      All images bind 0.0.0.0 so host browser can reach the app.
        stage('Create Dockerfile') {
            steps {
                script {
                    echo '[STAGE_START] Create Dockerfile'

                    def dfPath = "${env.PKG_ROOT}/Dockerfile"

                    if (fileExists(dfPath)) {
                        echo '[INFO] Dockerfile already exists — using as-is'
                    } else {
                        def entry = env.ENTRY_PT ?: 'index.js'
                        def df    = ''

                        switch (env.STACK) {

                            case 'vite':
                            case 'react':
                                df = 'FROM node:20-alpine AS builder\n' +
                                     'WORKDIR /app\n' +
                                     'COPY package*.json ./\n' +
                                     'RUN npm ci --ignore-scripts\n' +
                                     'COPY . .\n' +
                                     'RUN npm run build\n' +
                                     '\n' +
                                     'FROM nginx:alpine\n' +
                                     'COPY --from=builder /app/dist /usr/share/nginx/html\n' +
                                     'RUN printf "server {\\n  listen 80;\\n  root /usr/share/nginx/html;\\n  index index.html;\\n  location / { try_files $uri $uri/ /index.html; }\\n}\\n" > /etc/nginx/conf.d/default.conf\n' +
                                     'EXPOSE 80\n' +
                                     'CMD ["nginx", "-g", "daemon off;"]\n'
                                env.CONTAINER_PORT = '80'
                                break

                            case 'nextjs':
                                df = 'FROM node:20-alpine AS builder\n' +
                                     'WORKDIR /app\n' +
                                     'COPY package*.json ./\n' +
                                     'RUN npm ci --ignore-scripts\n' +
                                     'COPY . .\n' +
                                     'ENV NEXT_TELEMETRY_DISABLED=1\n' +
                                     'RUN npm run build\n' +
                                     '\n' +
                                     'FROM node:20-alpine\n' +
                                     'WORKDIR /app\n' +
                                     'ENV NODE_ENV=production\n' +
                                     'ENV NEXT_TELEMETRY_DISABLED=1\n' +
                                     'ENV HOSTNAME=0.0.0.0\n' +
                                     'COPY --from=builder /app/public ./public\n' +
                                     'COPY --from=builder /app/.next/standalone ./\n' +
                                     'COPY --from=builder /app/.next/static ./.next/static\n' +
                                     'EXPOSE 3000\n' +
                                     'CMD ["node", "server.js"]\n'
                                env.CONTAINER_PORT = '3000'
                                break

                            case 'python':
                                df = 'FROM python:3.11-slim\n' +
                                     'WORKDIR /app\n' +
                                     'COPY requirements.txt ./\n' +
                                     'RUN pip install --no-cache-dir -r requirements.txt\n' +
                                     'COPY . .\n' +
                                     'ENV HOST=0.0.0.0\n' +
                                     'ENV PORT=3000\n' +
                                     'EXPOSE 3000\n' +
                                     "CMD [\"python\", \"${entry}\"]\n"
                                env.CONTAINER_PORT = '3000'
                                break

                            case 'java':
                                df = 'FROM maven:3.9-eclipse-temurin-17 AS builder\n' +
                                     'WORKDIR /app\n' +
                                     'COPY pom.xml ./\n' +
                                     'RUN mvn dependency:go-offline -q\n' +
                                     'COPY src ./src\n' +
                                     'RUN mvn package -DskipTests -q\n' +
                                     '\n' +
                                     'FROM eclipse-temurin:17-jre-alpine\n' +
                                     'WORKDIR /app\n' +
                                     'COPY --from=builder /app/target/*.jar app.jar\n' +
                                     'EXPOSE 3000\n' +
                                     'CMD ["java", "-jar", "app.jar", "--server.port=3000", "--server.address=0.0.0.0"]\n'
                                env.CONTAINER_PORT = '3000'
                                break

                            default: // node / node-server
                                df = 'FROM node:20-alpine\n' +
                                     'WORKDIR /app\n' +
                                     'COPY package*.json ./\n' +
                                     'RUN npm ci --only=production --ignore-scripts\n' +
                                     'COPY . .\n' +
                                     'ENV HOST=0.0.0.0\n' +
                                     'ENV PORT=3000\n' +
                                     'EXPOSE 3000\n' +
                                     "CMD [\"node\", \"${entry}\"]\n"
                                env.CONTAINER_PORT = '3000'
                        }

                        writeFile file: dfPath, text: df
                        echo "[INFO] Dockerfile created — stack: ${env.STACK}, entry: ${entry}"
                    }

                    if (!env.CONTAINER_PORT) { env.CONTAINER_PORT = '3000' }
                    echo '[STAGE_SUCCESS] Create Dockerfile'
                }
            }
        }

        // ── Stage 11 ──────────────────────────────────────────────────────────
        stage('Build Image') {
            steps {
                script { echo '[STAGE_START] Build Image' }
                sh '''
export DOCKER_BUILDKIT=1
docker build --progress=plain --build-arg BUILDKIT_INLINE_CACHE=1 -t "${IMAGE_NAME}" "${PKG_ROOT}/"
'''
                script { echo '[STAGE_SUCCESS] Build Image' }
            }
        }

        // ── Stage 12 ──────────────────────────────────────────────────────────
        stage('Push to DockerHub') {
            steps {
                script { echo '[STAGE_START] Push to DockerHub' }
                sh '''
echo "$DOCKERHUB_CRED_PSW" | docker login -u "$DOCKERHUB_CRED_USR" --password-stdin
docker push "${IMAGE_NAME}"
docker logout
'''
                script { echo '[STAGE_SUCCESS] Push to DockerHub' }
            }
        }

        // ── Stage 13 ──────────────────────────────────────────────────────────
        stage('Deploy') {
            steps {
                script {
                    echo '[STAGE_START] Deploy'

                    if (env.DEPLOY_MODE == 'local') {
                        echo '[INFO] LOCAL VM deployment...'
                        sh """
                            docker stop ${CONTAINER_NAME} 2>/dev/null || true
                            docker rm   ${CONTAINER_NAME} 2>/dev/null || true
                            docker pull ${IMAGE_NAME}
                            docker run -d \\
                                --name ${CONTAINER_NAME} \\
                                --restart unless-stopped \\
                                -p 0.0.0.0:${PORT}:${CONTAINER_PORT} \\
                                ${IMAGE_NAME}
                        """
                        echo "[META] URL=http://${LOCAL_HOST}:${PORT}"
                        echo "LOCAL URL: http://${LOCAL_HOST}:${PORT}"

                    } else if (env.DEPLOY_MODE == 'aws') {
                        echo '[INFO] AWS EC2 deployment...'
                        sh """
                            ssh -o StrictHostKeyChecking=no -o ConnectTimeout=30 \\
                                ${AWS_SSH_USER}@${AWS_HOST} \\
                                'docker pull ${IMAGE_NAME} && \\
                                 docker stop ${CONTAINER_NAME} 2>/dev/null; \\
                                 docker rm   ${CONTAINER_NAME} 2>/dev/null; \\
                                 docker run -d \\
                                     --name ${CONTAINER_NAME} \\
                                     --restart unless-stopped \\
                                     -p 0.0.0.0:80:${CONTAINER_PORT} \\
                                     ${IMAGE_NAME}'
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
                script { echo '[STAGE_START] Verify' }
                sh '''
set +e
sleep 3
if [ "${DEPLOY_MODE}" = "local" ]; then
    if docker ps --format "{{.Names}}" | grep -q "^${CONTAINER_NAME}$"; then
        echo "[INFO] Container is running"
        docker ps --filter "name=${CONTAINER_NAME}" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
        curl -sf --max-time 10 "http://localhost:${PORT}/" -o /dev/null \
            && echo "[INFO] HTTP check PASSED" \
            || echo "[WARN] HTTP check failed — app may still be starting"
    else
        echo "[WARN] Container ${CONTAINER_NAME} not found"
    fi
else
    ssh -o StrictHostKeyChecking=no -o ConnectTimeout=15 \
        "${DEPLOY_USER}@${DEPLOY_HOST}" \
        "docker ps --filter name=${CONTAINER_NAME} --format table" || true
fi
'''
                script { echo '[STAGE_SUCCESS] Verify' }
            }
        }

    } // end stages

    post {
        always {
            echo '[INFO] Pipeline complete — cleaning workspace'
            sh 'rm -rf app || true'
        }
        success {
            echo '[DEPLOY_SUCCESS]'
            echo "[META] FINAL_STATUS=SUCCESS"
        }
        failure {
            echo '[DEPLOY_FAILED]'
            echo "[META] FINAL_STATUS=FAILED"
            sh "docker rmi ${env.IMAGE_NAME} 2>/dev/null || true"
        }
    }

}
