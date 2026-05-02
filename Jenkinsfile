pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL',      defaultValue: '', description: 'GitHub repo URL to deploy')
        string(name: 'APP_NAME',      defaultValue: '', description: 'Unique app name')
        choice(name: 'DEPLOY_TARGET', choices: ['local', 'aws'], description: 'local VM or AWS EC2')
    }

    environment {
        DOCKERHUB_CRED = credentials('dockerhub-cred')
        DOCKERHUB_USER = 'raheeljamal'
        LOCAL_HOST     = '192.168.122.127'
        LOCAL_SSH_USER = 'ubuntu'
        AWS_SSH_USER   = 'ubuntu'
        AWS_HOST       = "${env.AWS_EC2_IP ?: 'YOUR_EC2_PUBLIC_IP'}"
    }

    stages {

        stage('Init') {
            steps {
                script {
                    echo "[INFO] DEPLOY_TARGET: ${params.DEPLOY_TARGET}"
                    echo "[INFO] REPO_URL: ${params.REPO_URL}"
                    echo "[INFO] APP_NAME: ${params.APP_NAME}"
                }
            }
        }

        stage('Input Repo') {
            steps {
                script {
                    def repoUrl = params.REPO_URL?.trim()
                    def appName = params.APP_NAME?.trim()
                    if (!repoUrl) error('REPO_URL is required')
                    if (!appName) error('APP_NAME is required')

                    def appId = sh(script: "printf '%s' '${repoUrl}' | md5sum | cut -c1-6", returnStdout: true).trim()

                    env.REPO_URL       = repoUrl
                    env.APP_NAME       = appName
                    env.APP_ID         = appId
                    env.CONTAINER_NAME = "app_${appId}"
                    env.IMAGE_NAME     = "${DOCKERHUB_USER}/app_${appId}:latest"
                    env.PKG_ROOT       = 'app'
                    env.CONTAINER_PORT = '3000'

                    echo "[META] APP_ID=${appId} | IMAGE=${env.IMAGE_NAME}"
                }
            }
        }

        stage('Select Deploy Mode') {
            steps {
                script {
                    if (params.DEPLOY_TARGET == 'local') {
                        env.DEPLOY_MODE = 'local'
                        env.DEPLOY_HOST = env.LOCAL_HOST
                        env.DEPLOY_USER = env.LOCAL_SSH_USER
                    } else {
                        env.DEPLOY_MODE = 'aws'
                        env.DEPLOY_HOST = env.AWS_HOST
                        env.DEPLOY_USER = env.AWS_SSH_USER
                    }
                    echo "[META] DEPLOY_MODE=${env.DEPLOY_MODE} | HOST=${env.DEPLOY_HOST}"
                }
            }
        }

        stage('Allocate Port') {
            steps {
                script {
                    if (env.DEPLOY_MODE == 'local') {
                        def used = sh(script: "docker ps --format '{{.Ports}}' | grep -oP '[0-9]+(?=->)' || true", returnStdout: true).trim()
                        def usedList = used ? used.split('\n').collect { it.trim() } : []
                        def port = 3000
                        while (usedList.contains(port.toString())) { port++ }
                        env.PORT = port.toString()
                    } else {
                        env.PORT = '80'
                    }
                    echo "[META] PORT=${env.PORT}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                sh 'rm -rf app && git clone --depth=1 "${REPO_URL}" app'
            }
        }

        stage('Setup Docker Ignore') {
            steps {
                script {
                    // writeFile never touches the shell — zero quoting issues
                    writeFile file: 'app/.dockerignore', text: '''\
.git
.gitignore
node_modules
bower_components
dist
build
out
.next
.nuxt
.cache
coverage
.nyc_output
__tests__
*.test.js
*.spec.js
*.test.ts
*.spec.ts
jest.config.*
cypress
playwright
.eslintrc*
.prettierrc*
.husky
*.log
logs
.env
.env.*
*.pem
*.key
*.cert
secrets
.DS_Store
Thumbs.db
Dockerfile*
docker-compose*
.dockerignore
.github
.circleci
docs
*.md
README*
LICENSE*
__pycache__
*.pyc
.venv
venv
target
.gradle
*.class
*.jar
*.war
.terraform
.serverless
'''
                }
            }
        }

        stage('Secret Scan') {
            steps {
                script {
                    // FIX: use sh("") double-quote block so Groovy handles escaping.
                    // grep -r needs no backslash parens — safe in all block types.
                    def result = sh(
                        script: 'grep -rniE "(password|api_key|secret|token)\\s*=\\s*.{8,}" app --include="*.js" --include="*.ts" --include="*.py" --include="*.env" --exclude-dir=node_modules --exclude-dir=.git || true',
                        returnStdout: true
                    ).trim()
                    if (result) {
                        echo "[WARN] Potential secrets found:"
                        echo result
                        // Uncomment below to block the build on secrets found:
                        // error('Secret scan failed')
                    } else {
                        echo "[META] SECRET_SCAN=PASSED"
                    }
                }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    def stack   = 'node'
                    def pkgRoot = 'app'
                    def entry   = 'index.js'

                    // Monorepo probe
                    for (sub in ['packages','apps','services','frontend','backend','client','server','api','web']) {
                        if (fileExists("app/${sub}/package.json")) {
                            pkgRoot = "app/${sub}"
                            echo "[INFO] Monorepo sub-dir: ${sub}"
                            break
                        }
                    }

                    if (fileExists("${pkgRoot}/package.json")) {
                        def pkg = readFile("${pkgRoot}/package.json")
                        if      (pkg.contains('"vite"'))    stack = 'vite'
                        else if (pkg.contains('"next"'))    stack = 'nextjs'
                        else if (pkg.contains('"react"'))   stack = 'react'
                        else if (pkg.contains('"express"') || pkg.contains('"fastify"') || pkg.contains('"koa"')) stack = 'node-server'
                        else                                stack = 'node'

                        // Detect entry point
                        def m = (pkg =~ /"main"\s*:\s*"([^"]+)"/)
                        if (m.find()) {
                            entry = m.group(1)
                        } else {
                            for (ep in ['index.js','server.js','app.js','main.js','src/index.js','src/server.js']) {
                                if (fileExists("${pkgRoot}/${ep}")) { entry = ep; break }
                            }
                        }
                    } else if (fileExists("${pkgRoot}/requirements.txt")) {
                        stack = 'python'
                        for (ep in ['app.py','main.py','run.py','server.py']) {
                            if (fileExists("${pkgRoot}/${ep}")) { entry = ep; break }
                        }
                    } else if (fileExists("${pkgRoot}/pom.xml")) {
                        stack = 'java'; entry = ''
                    }

                    env.STACK    = stack
                    env.PKG_ROOT = pkgRoot
                    env.ENTRY_PT = entry
                    echo "[META] STACK=${stack} | PKG_ROOT=${pkgRoot} | ENTRY=${entry}"
                }
            }
        }

        stage('Dependency Audit') {
            steps {
                script {
                    if (fileExists("${env.PKG_ROOT}/package.json")) {
                        // sh("") double-quote block — Groovy-interpolated, no bare backslashes needed
                        sh "docker run --rm -v \"\$(pwd)/${env.PKG_ROOT}:/work\" -w /work node:20-alpine sh -c 'npm install --prefer-offline --ignore-scripts 2>&1 | tail -5 && npm audit --json 2>/dev/null || true' > /tmp/audit.txt 2>&1 || true"
                        def out = readFile('/tmp/audit.txt')
                        def crit = (out =~ /"critical":(\d+)/) ? (out =~ /"critical":(\d+)/)[0][1] : '0'
                        def high = (out =~ /"high":(\d+)/)     ? (out =~ /"high":(\d+)/)[0][1]     : '0'
                        echo "[META] VULN_CRITICAL=${crit} | VULN_HIGH=${high}"
                    } else {
                        echo "[INFO] No package.json — skipping audit"
                    }
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    def dfPath = "${env.PKG_ROOT}/Dockerfile"
                    if (fileExists(dfPath)) {
                        echo '[INFO] Dockerfile exists in repo — using as-is'
                        // Detect port from existing Dockerfile
                        def dfContent = readFile(dfPath)
                        if (dfContent.contains('EXPOSE 80')) { env.CONTAINER_PORT = '80' }
                    } else {
                        def entry = env.ENTRY_PT ?: 'index.js'
                        def df = ''

                        switch (env.STACK) {
                            case 'vite':
                            case 'react':
                                df = 'FROM node:20-alpine AS builder\n' +
                                     'WORKDIR /app\n' +
                                     'COPY package*.json ./\n' +
                                     'RUN npm ci --ignore-scripts\n' +
                                     'COPY . .\n' +
                                     'RUN npm run build\n\n' +
                                     'FROM nginx:alpine\n' +
                                     'COPY --from=builder /app/dist /usr/share/nginx/html\n' +
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
                                     'RUN npm run build\n\n' +
                                     'FROM node:20-alpine\n' +
                                     'WORKDIR /app\n' +
                                     'ENV NODE_ENV=production\n' +
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
                                     'RUN mvn package -DskipTests -q\n\n' +
                                     'FROM eclipse-temurin:17-jre-alpine\n' +
                                     'WORKDIR /app\n' +
                                     'COPY --from=builder /app/target/*.jar app.jar\n' +
                                     'EXPOSE 3000\n' +
                                     'CMD ["java", "-jar", "app.jar", "--server.port=3000", "--server.address=0.0.0.0"]\n'
                                env.CONTAINER_PORT = '3000'
                                break
                            default:
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
                        echo "[INFO] Dockerfile created — stack: ${env.STACK}"
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                sh 'DOCKER_BUILDKIT=1 docker build --progress=plain -t "${IMAGE_NAME}" "${PKG_ROOT}/"'
            }
        }

        stage('Push to DockerHub') {
            steps {
                sh 'echo "$DOCKERHUB_CRED_PSW" | docker login -u "$DOCKERHUB_CRED_USR" --password-stdin && docker push "${IMAGE_NAME}" && docker logout'
            }
        }

        stage('Deploy') {
            steps {
                script {
                    if (env.DEPLOY_MODE == 'local') {
                        sh """
                            docker stop ${env.CONTAINER_NAME} 2>/dev/null || true
                            docker rm   ${env.CONTAINER_NAME} 2>/dev/null || true
                            docker pull ${env.IMAGE_NAME}
                            docker run -d --name ${env.CONTAINER_NAME} --restart unless-stopped -p 0.0.0.0:${env.PORT}:${env.CONTAINER_PORT} ${env.IMAGE_NAME}
                        """
                        echo "APP URL: http://${env.LOCAL_HOST}:${env.PORT}"
                    } else {
                        sh """
                            ssh -o StrictHostKeyChecking=no -o ConnectTimeout=30 ${env.AWS_SSH_USER}@${env.AWS_HOST} 'docker pull ${env.IMAGE_NAME} && docker stop ${env.CONTAINER_NAME} 2>/dev/null; docker rm ${env.CONTAINER_NAME} 2>/dev/null; docker run -d --name ${env.CONTAINER_NAME} --restart unless-stopped -p 0.0.0.0:80:${env.CONTAINER_PORT} ${env.IMAGE_NAME}'
                        """
                        echo "APP URL: http://${env.AWS_HOST}"
                    }
                }
            }
        }

        stage('Verify') {
            steps {
                script {
                    sleep 3
                    if (env.DEPLOY_MODE == 'local') {
                        def running = sh(script: "docker ps --format '{{.Names}}' | grep -c '${env.CONTAINER_NAME}' || true", returnStdout: true).trim()
                        if (running == '1') {
                            echo "[OK] Container ${env.CONTAINER_NAME} is running"
                            sh "docker ps --filter name=${env.CONTAINER_NAME} --format 'table {{.Names}}\\t{{.Status}}\\t{{.Ports}}'"
                            sh "curl -sf --max-time 10 http://localhost:${env.PORT}/ -o /dev/null && echo 'HTTP OK' || echo 'HTTP not ready yet'"
                        } else {
                            echo "[WARN] Container not found — check docker logs"
                        }
                    } else {
                        sh "ssh -o StrictHostKeyChecking=no -o ConnectTimeout=15 ${env.DEPLOY_USER}@${env.DEPLOY_HOST} 'docker ps --filter name=${env.CONTAINER_NAME}' || true"
                    }
                }
            }
        }

    } // end stages

    post {
        always {
            sh 'rm -rf app || true'
            echo '[INFO] Workspace cleaned'
        }
        success {
            echo "[DEPLOY_SUCCESS] ${env.IMAGE_NAME} running on port ${env.PORT}"
        }
        failure {
            echo '[DEPLOY_FAILED]'
            sh "docker rmi ${env.IMAGE_NAME} 2>/dev/null || true"
        }
    }
}
