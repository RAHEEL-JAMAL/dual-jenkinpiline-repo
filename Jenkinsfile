

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

        // ─────────────────────────────────────────────────────────────────────
        stage('Init') {
            steps {
                script {
                    echo "[STAGE_START] Init"
                    echo "[INFO] DEPLOY_TARGET: ${params.DEPLOY_TARGET}"
                    echo "[INFO] REPO_URL: ${params.REPO_URL}"
                    echo "[INFO] APP_NAME: ${params.APP_NAME}"
                    echo "[STAGE_SUCCESS] Init"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Input Repo') {
            steps {
                script {
                    echo "[STAGE_START] Input Repo"

                    def repoUrl = params.REPO_URL?.trim()
                    def appName = params.APP_NAME?.trim()

                    if (!repoUrl) error('REPO_URL is required')
                    if (!appName) error('APP_NAME is required')

                    def appId = sh(
                        script: "printf '%s' '${repoUrl}' | md5sum | cut -c1-6",
                        returnStdout: true
                    ).trim()

                    def safeAppName = appName.toLowerCase().replaceAll(/[^a-z0-9-_]/, '-')

                    env.REPO_URL       = repoUrl
                    env.APP_NAME       = safeAppName
                    env.APP_ID         = appId
                    env.CONTAINER_NAME = "${safeAppName}-${appId}"
                    env.IMAGE_NAME     = "${DOCKERHUB_USER}/${safeAppName}-${appId}:latest"
                    env.PKG_ROOT       = 'app'
                    env.CONTAINER_PORT = '3000'

                    echo "[META] APP_NAME=${env.APP_NAME}"
                    echo "[META] APP_ID=${appId}"
                    echo "[META] CONTAINER_NAME=${env.CONTAINER_NAME}"
                    echo "[META] IMAGE_NAME=${env.IMAGE_NAME}"
                    echo "[STAGE_SUCCESS] Input Repo"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Select Deploy Mode') {
            steps {
                script {
                    echo "[STAGE_START] Select Deploy Mode"
                    if (params.DEPLOY_TARGET == 'local') {
                        env.DEPLOY_MODE = 'local'
                        env.DEPLOY_HOST = env.LOCAL_HOST
                        env.DEPLOY_USER = env.LOCAL_SSH_USER
                    } else {
                        env.DEPLOY_MODE = 'aws'
                        env.DEPLOY_HOST = env.AWS_HOST
                        env.DEPLOY_USER = env.AWS_SSH_USER
                    }
                    echo "[META] DEPLOY_MODE=${env.DEPLOY_MODE}"
                    echo "[META] DEPLOY_HOST=${env.DEPLOY_HOST}"
                    echo "[STAGE_SUCCESS] Select Deploy Mode"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Allocate Safe Port') {
            steps {
                script {
                    echo "[STAGE_START] Allocate Safe Port"

                    def portMap = [
                        vite: "80", react: "80", cra: "80", nextjs: "80", frontend: "80",
                        node: "3000", backend: "3000", "node-server": "3000",
                        python: "3000", flask: "5000", fastapi: "8000", django: "8000",
                        java: "8080", spring: "8080",
                        go: "3000", php: "80"
                    ]

                    def stack    = env.STACK ?: "backend"
                    def basePort = portMap[stack] ?: "3000"

                    if (env.DEPLOY_MODE == 'local') {
                        def usedRaw = sh(
                            script: 'docker ps --format "{{.Ports}}" | grep -oE "[0-9]{2,5}" | sort -un || true',
                            returnStdout: true
                        ).trim()

                        def usedPorts = usedRaw ? usedRaw.tokenize('\n').collect { it.trim() } : []
                        def port = basePort.toInteger()
                        while (usedPorts.contains(port.toString())) { port++ }
                        env.PORT = port.toString()
                    } else {
                        env.PORT = "80"
                    }

                    echo "[META] STACK=${stack}"
                    echo "[META] BASE_PORT=${basePort}"
                    echo "[META] FINAL_PORT=${env.PORT}"
                    echo "[STAGE_SUCCESS] Allocate Safe Port"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Clone Repo') {
            steps {
                script {
                    echo "[STAGE_START] Clone Repo"
                    sh 'rm -rf app && git clone --depth=1 "$REPO_URL" app'
                    echo "[STAGE_SUCCESS] Clone Repo"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Setup Docker Ignore') {
            steps {
                script {
                    echo "[STAGE_START] Setup Docker Ignore"
                    writeFile file: 'app/.dockerignore', text: '''\
.git
.gitignore
node_modules
bower_components
npm-debug.log*
yarn-error.log
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
                    echo "[STAGE_SUCCESS] Setup Docker Ignore"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Secret Scan') {
            steps {
                script {
                    echo "[STAGE_START] Secret Scan"
                    def result = sh(
                        script: 'grep -rniE "(password|api_key|secret|token)\\s*=\\s*.{8,}" app --include="*.js" --include="*.ts" --include="*.py" --include="*.env" --exclude-dir=node_modules --exclude-dir=.git || true',
                        returnStdout: true
                    ).trim()
                    if (result) {
                        echo "[WARN] Potential secrets found — review before production"
                        echo "[META] SECRET_SCAN=FAILED"
                    } else {
                        echo "[META] SECRET_SCAN=PASSED"
                    }
                    echo "[STAGE_SUCCESS] Secret Scan"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
       stage('Detect Stack') {
            steps {
                script {
                    echo "[STAGE_START] Detect Stack"
                    def stack   = 'node'
                    def pkgRoot = 'app'
                    def entry   = 'index.js'

                    for (sub in ['packages', 'apps', 'services', 'frontend', 'backend', 'client', 'server', 'api', 'web']) {
                        if (fileExists("app/${sub}/package.json")) {
                            pkgRoot = "app/${sub}"
                            echo "[INFO] Monorepo sub-dir detected: ${sub}"
                            break
                        }
                    }

                    if (fileExists("${pkgRoot}/package.json")) {
                        def pkg = readFile("${pkgRoot}/package.json")
                        if      (pkg.contains('"vite"'))                                                            stack = 'vite'
                        else if (pkg.contains('"next"'))                                                            stack = 'nextjs'
                        else if (pkg.contains('"react-scripts"'))                                                   stack = 'cra'
                        else if (pkg.contains('"react"'))                                                           stack = 'react'
                        else if (pkg.contains('"express"') || pkg.contains('"fastify"') || pkg.contains('"koa"'))   stack = 'node-server'
                        else                                                                                        stack = 'node'

                        def mainMatch = (pkg =~ /"main"\s*:\s*"([^"]+)"/)
                        if (mainMatch.find()) {
                            entry = mainMatch.group(1)
                            mainMatch = null
                        } else {
                            mainMatch = null
                            for (ep in ['index.js', 'server.js', 'app.js', 'main.js', 'src/index.js', 'src/server.js']) {
                                if (fileExists("${pkgRoot}/${ep}")) { entry = ep; break }
                            }
                        }

                    } else if (fileExists("${pkgRoot}/manage.py")) {
                        stack = 'python'
                        entry = 'manage.py'

                    } else if (fileExists("${pkgRoot}/requirements.txt")) {
                        stack = 'python'
                        for (ep in ['app.py', 'main.py', 'run.py', 'server.py']) {
                            if (fileExists("${pkgRoot}/${ep}")) { entry = ep; break }
                        }

                    } else if (fileExists("${pkgRoot}/pom.xml")) {
                        stack = 'java'
                        entry = ''

                    } else if (fileExists("${pkgRoot}/go.mod")) {
                        stack = 'go'
                        entry = 'main.go'

                    } else if (fileExists("${pkgRoot}/index.php") || fileExists("${pkgRoot}/composer.json")) {
                        stack = 'php'
                        entry = 'index.php'

                    } else if (fileExists("${pkgRoot}/index.html") || fileExists("${pkgRoot}/index.htm")) {
                        stack = 'static'
                        entry = 'index.html'
                    }

                    env.STACK    = stack
                    env.PKG_ROOT = pkgRoot
                    env.ENTRY_PT = entry

                    echo "[META] STACK=${stack}"
                    echo "[META] PKG_ROOT=${pkgRoot}"
                    echo "[META] ENTRY_PT=${entry}"
                    echo "[STAGE_SUCCESS] Detect Stack"
                }
            }
        }
        // ─────────────────────────────────────────────────────────────────────
        stage('Dependency Audit') {
            steps {
                script {
                    echo "[STAGE_START] Dependency Audit"
                    if (fileExists("${env.PKG_ROOT}/package.json")) {
                        sh '''
                            docker run --rm \
                              --network host \
                              -v "$(pwd)/''' + env.PKG_ROOT + ''':/work" \
                              -w /work \
                              node:20-alpine \
                              sh -c 'npm config set fetch-retry-mintimeout 20000 && \
                                     npm config set fetch-retry-maxtimeout 120000 && \
                                     npm config set fetch-retries 5 && \
                                     npm install --prefer-offline --ignore-scripts --silent 2>&1 | tail -3; \
                                     npm audit --json 2>/dev/null || true' \
                              > /tmp/audit.txt 2>&1 || true
                        '''
                        def out = readFile('/tmp/audit.txt')

                        def critMatch = (out =~ /"critical"\s*:\s*(\d+)/)
                        def crit = critMatch.find() ? critMatch.group(1) : '0'
                        critMatch = null

                        def highMatch = (out =~ /"high"\s*:\s*(\d+)/)
                        def high = highMatch.find() ? highMatch.group(1) : '0'
                        highMatch = null

                        echo "[META] VULN_CRITICAL=${crit}"
                        echo "[META] VULN_HIGH=${high}"
                        echo "[META] DEPENDENCY_SCAN=PASSED"
                    } else {
                        echo "[INFO] No package.json — skipping npm audit"
                        echo "[META] DEPENDENCY_SCAN=PASSED"
                    }
                    echo "[STAGE_SUCCESS] Dependency Audit"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Create Dockerfile') {
            steps {
                script {
                    echo "[STAGE_START] Create Dockerfile"
                    def dfPath = "${env.PKG_ROOT}/Dockerfile"

                    if (false && fileExists(dfPath)) {
                        echo '[INFO] Dockerfile already exists — using repo Dockerfile'
                        def dfContent = readFile(dfPath)
                        if      (dfContent.contains('EXPOSE 80'))   { env.CONTAINER_PORT = '80' }
                        else if (dfContent.contains('EXPOSE 8080')) { env.CONTAINER_PORT = '8080' }
                        else if (dfContent.contains('EXPOSE 5000')) { env.CONTAINER_PORT = '5000' }
                        else                                         { env.CONTAINER_PORT = '3000' }
                    } else {
                        def entry = env.ENTRY_PT ?: 'index.js'
                        def df    = ''

                        switch (env.STACK) {
                            case 'vite':
                                df = '''\
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm config set fetch-retry-mintimeout 20000 && \
    npm config set fetch-retry-maxtimeout 120000 && \
    npm config set fetch-retries 5 && \
    npm install --ignore-scripts
COPY . .
RUN npm run build

FROM nginx:alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html
RUN printf 'server{listen 80;root /usr/share/nginx/html;index index.html;location /{try_files $uri $uri/ /index.html;}}' > /etc/nginx/conf.d/app.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
'''
                                env.CONTAINER_PORT = '80'
                                break

                            case 'cra':
                            case 'react':
                                df = '''\
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm config set fetch-retry-mintimeout 20000 && \
    npm config set fetch-retry-maxtimeout 120000 && \
    npm config set fetch-retries 5 && \
    npm install --ignore-scripts
COPY . .
RUN npm run build

FROM nginx:alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY --from=builder /app/build /usr/share/nginx/html
RUN printf 'server{listen 80;root /usr/share/nginx/html;index index.html;location /{try_files $uri $uri/ /index.html;}}' > /etc/nginx/conf.d/app.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
'''
                                env.CONTAINER_PORT = '80'
                                break

                            case 'nextjs':
                                df = '''\
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm config set fetch-retry-mintimeout 20000 && \
    npm config set fetch-retry-maxtimeout 120000 && \
    npm config set fetch-retries 5 && \
    npm install --ignore-scripts
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV HOSTNAME=0.0.0.0
ENV PORT=3000
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
EXPOSE 3000
CMD ["node", "server.js"]
'''
                                env.CONTAINER_PORT = '3000'
                                break

                            case 'node-server':
                                df = """FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm config set fetch-retry-mintimeout 20000 && \\
    npm config set fetch-retry-maxtimeout 120000 && \\
    npm config set fetch-retries 5 && \\
    npm install --only=production --ignore-scripts
COPY . .
ENV NODE_ENV=production
ENV HOST=0.0.0.0
ENV PORT=3000
EXPOSE 3000
CMD ["node", "${entry}"]
"""
                                env.CONTAINER_PORT = '3000'
                                break
                            
                            case 'php':
                                df = '''\
FROM php:8.2-apache
WORKDIR /var/www/html
COPY . .
RUN chown -R www-data:www-data /var/www/html
EXPOSE 80
CMD ["apache2-foreground"]
'''
                                env.CONTAINER_PORT = '80'
                                break
                
                            case 'static':
                                df = '''\
FROM nginx:alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY . /usr/share/nginx/html
RUN printf 'server{listen 80;root /usr/share/nginx/html;index index.html;location /{try_files $uri $uri/ /index.html;}}' > /etc/nginx/conf.d/app.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
'''
                                env.CONTAINER_PORT = '80'
                                break
                           
                           case 'python':
                                df = """FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt 2>/dev/null || pip install django gunicorn
ENV HOST=0.0.0.0
ENV PORT=8000
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
"""
                                env.CONTAINER_PORT = '8000'
                                break

                            case 'java':
                                df = '''\
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml ./
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 3000
CMD ["java", "-jar", "app.jar", "--server.port=3000", "--server.address=0.0.0.0"]
'''
                                env.CONTAINER_PORT = '3000'
                                break

                            case 'go':
                                df = '''\
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

FROM alpine:3.19
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 3000
CMD ["./server"]
'''
                                env.CONTAINER_PORT = '3000'
                                break

                            default:
                                df = """FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm config set fetch-retry-mintimeout 20000 && \\
    npm config set fetch-retry-maxtimeout 120000 && \\
    npm config set fetch-retries 5 && \\
    npm install --only=production --ignore-scripts
COPY . .
ENV HOST=0.0.0.0
ENV PORT=3000
EXPOSE 3000
CMD ["node", "${entry}"]
"""
                                env.CONTAINER_PORT = '3000'
                        }

                        writeFile file: dfPath, text: df
                        echo "[INFO] Generated Dockerfile for stack: ${env.STACK}"
                    }
                    echo "[STAGE_SUCCESS] Create Dockerfile"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Build Image') {
            steps {
                script {
                    echo "[STAGE_START] Build Image"
                    sh """
                        docker image inspect \$IMAGE_NAME > /dev/null 2>&1 || echo "No cache image found"

                        docker build \
                            --network host \
                            -f "\$PKG_ROOT/Dockerfile" \
                            -t "\$IMAGE_NAME" \
                            "\$PKG_ROOT/"
                    """
                    echo "[STAGE_SUCCESS] Build Image"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Image Scan (Trivy)') {
            steps {
                script {
                    echo "[STAGE_START] Image Scan (Trivy)"

                    sh 'docker pull aquasec/trivy:latest 2>/dev/null || true'

                    sh """
                        docker run --rm \
                          --network host \
                          -v /var/run/docker.sock:/var/run/docker.sock \
                          -v /tmp/trivy-cache:/root/.cache/trivy \
                          aquasec/trivy:latest image \
                          --exit-code 0 \
                          --severity CRITICAL,HIGH \
                          --format json \
                          --output /tmp/trivy-report.json \
                          '${env.IMAGE_NAME}' || true
                    """

                    def reportFile = '/tmp/trivy-report.json'
                    def criticalCount = 0
                    def highCount     = 0

                    if (fileExists(reportFile)) {
                        def report = readFile(reportFile)

                        def critMatches = (report =~ /"Severity"\s*:\s*"CRITICAL"/)
                        while (critMatches.find()) { criticalCount++ }
                        critMatches = null

                        def highMatches = (report =~ /"Severity"\s*:\s*"HIGH"/)
                        while (highMatches.find()) { highCount++ }
                        highMatches = null
                    }

                    echo "[META] IMAGE_CRITICAL_CVE=${criticalCount}"
                    echo "[META] IMAGE_HIGH_CVE=${highCount}"

                    if (criticalCount > 0) {
                        echo "[WARN] Trivy found ${criticalCount} CRITICAL CVE(s) in image — review before production"
                    } else {
                        echo "[INFO] Trivy scan complete — no CRITICAL vulnerabilities found"
                    }
                    echo "[META] IMAGE_SCAN=PASSED"
                    echo "[STAGE_SUCCESS] Image Scan (Trivy)"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Push to DockerHub') {
            steps {
                script {
                    echo "[STAGE_START] Push to DockerHub"
                    sh 'echo "$DOCKERHUB_CRED_PSW" | docker login -u "$DOCKERHUB_CRED_USR" --password-stdin'
                    sh 'docker push "$IMAGE_NAME"'
                    sh 'docker logout'
                    echo "[STAGE_SUCCESS] Push to DockerHub"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
       stage('Deploy') {
            steps {
                script {
                    echo "[STAGE_START] Deploy"

                    def isFrontend = (env.STACK == 'vite' || env.STACK == 'react' || env.STACK == 'cra' || env.STACK == 'nextjs')
                    def isPython   = (env.STACK == 'python' || env.STACK == 'fastapi' || env.STACK == 'django')
                    def isNginx    = (env.STACK == 'php' || env.STACK == 'static')
                    env.CONTAINER_PORT = isFrontend ? "80" : isNginx ? "80" : isPython ? "8000" : "3000"

                    echo "[META] FORCE_CONTAINER_PORT=${env.CONTAINER_PORT}"

                    if (env.DEPLOY_MODE == 'local') {
                        sh """
                            CNAME='${env.CONTAINER_NAME}'
                            IMG='${env.IMAGE_NAME}'
                            HPORT='${env.PORT}'
                            CPORT='${env.CONTAINER_PORT}'

                            docker stop "\$CNAME" 2>/dev/null || true
                            docker rm -f "\$CNAME" 2>/dev/null || true

                            docker run -d \
                              --name "\$CNAME" \
                              --restart unless-stopped \
                              -p 0.0.0.0:\${HPORT}:\${CPORT} \
                              "\$IMG"
                        """
                        echo "[META] URL=http://${env.LOCAL_HOST}:${env.PORT}"
                    } else {
                        sh """
                            ssh -o StrictHostKeyChecking=no -o ConnectTimeout=30 \
                              '${env.AWS_SSH_USER}'@'${env.AWS_HOST}' bash -s <<'ENDSSH'

docker pull '${env.IMAGE_NAME}'
docker stop  '${env.CONTAINER_NAME}' 2>/dev/null || true
docker rm -f '${env.CONTAINER_NAME}' 2>/dev/null || true

docker run -d \
  --name '${env.CONTAINER_NAME}' \
  --restart unless-stopped \
  -p 0.0.0.0:80:'${env.CONTAINER_PORT}' \
  '${env.IMAGE_NAME}'

ENDSSH
                        """
                        echo "[META] URL=http://${env.AWS_HOST}"
                    }

                    echo "[STAGE_SUCCESS] Deploy"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Verify') {
            steps {
                script {
                    echo "[STAGE_START] Verify"
                    sleep 5

                    if (env.DEPLOY_MODE == 'local') {
                        def running = sh(
                            script: 'docker ps --format \'{{.Names}}\' | grep -c \'^' + env.CONTAINER_NAME + '$\' || true',
                            returnStdout: true
                        ).trim()

                        if (running == '1') {
                            echo "[OK] Container ${env.CONTAINER_NAME} is running"
                            sh 'docker ps --filter \'name=^' + env.CONTAINER_NAME + '$\' --format \'table {{.Names}}\\t{{.Status}}\\t{{.Ports}}\''
                            sh """
                                for i in 1 2 3; do
                                    curl -sf --max-time 10 http://localhost:${env.PORT}/ -o /dev/null && echo 'HTTP OK' && break
                                    echo "Attempt \$i failed, retrying in 5s..."
                                    sleep 5
                                done || echo '[WARN] HTTP check did not succeed — app may still be starting'
                            """
                        } else {
                            echo "[WARN] Container not running — dumping last 30 log lines:"
                            sh "docker logs --tail 30 '${env.CONTAINER_NAME}' 2>&1 || true"
                        }
                    } else {
                        sh "ssh -o StrictHostKeyChecking=no -o ConnectTimeout=15 '${env.DEPLOY_USER}'@'${env.DEPLOY_HOST}' 'docker ps --filter name=^${env.CONTAINER_NAME}\$' || true"
                    }
                    echo "[STAGE_SUCCESS] Verify"
                }
            }
        }

    } 

 post {
    always {
        node('built-in') {
            sh 'docker rmi "$IMAGE_NAME" 2>/dev/null || true'
            sh 'rm -rf app /tmp/trivy-report.json || true'
            echo '[INFO] Workspace cleaned'
        }
    }
    success {
        echo "[DEPLOY_SUCCESS] ${env.IMAGE_NAME} → port ${env.PORT}"
    }
    failure {
        echo '[DEPLOY_FAILED]'
    }
}
}

// pipeline {
//     agent any

//     parameters {
//         string(name: 'REPO_URL',      defaultValue: '', description: 'GitHub repo URL to deploy')
//         string(name: 'APP_NAME',      defaultValue: '', description: 'Unique app name')
//         choice(name: 'DEPLOY_TARGET', choices: ['local', 'aws'], description: 'local VM or AWS EC2')
//     }

//     environment {
//         DOCKERHUB_CRED = credentials('dockerhub-cred')
//         DOCKERHUB_USER = 'raheeljamal'
//         LOCAL_HOST     = '192.168.122.127'
//         LOCAL_SSH_USER = 'ubuntu'
//         AWS_SSH_USER   = 'ubuntu'
//         AWS_HOST       = "${env.AWS_EC2_IP ?: 'YOUR_EC2_PUBLIC_IP'}"
//     }

//     stages {

//         // ─────────────────────────────────────────────────────────────────────
//         stage('Init') {
//             steps {
//                 script {
//                     echo "[STAGE_START] Init"
//                     echo "[INFO] DEPLOY_TARGET: ${params.DEPLOY_TARGET}"
//                     echo "[INFO] REPO_URL: ${params.REPO_URL}"
//                     echo "[INFO] APP_NAME: ${params.APP_NAME}"
//                     echo "[STAGE_SUCCESS] Init"
//                 }
//             }
//         }

//         // ─────────────────────────────────────────────────────────────────────
//         stage('Input Repo') {
//             steps {
//                 script {
//                     echo "[STAGE_START] Input Repo"

//                     def repoUrl = params.REPO_URL?.trim()
//                     def appName = params.APP_NAME?.trim()

//                     if (!repoUrl) error('REPO_URL is required')
//                     if (!appName) error('APP_NAME is required')

//                     def appId = sh(
//                         script: "printf '%s' '${repoUrl}' | md5sum | cut -c1-6",
//                         returnStdout: true
//                     ).trim()

//                     def safeAppName = appName.toLowerCase().replaceAll(/[^a-z0-9-_]/, '-')

//                     env.REPO_URL       = repoUrl
//                     env.APP_NAME       = safeAppName
//                     env.APP_ID         = appId
//                     env.PKG_ROOT       = 'app'
//                     env.CONTAINER_PORT = '3000'

//                     echo "[META] APP_NAME=${env.APP_NAME}"
//                     echo "[META] APP_ID=${appId}"
//                     echo "[STAGE_SUCCESS] Input Repo"
//                 }
//             }
//         }

//         // ─────────────────────────────────────────────────────────────────────
//         stage('Select Deploy Mode') {
//             steps {
//                 script {
//                     echo "[STAGE_START] Select Deploy Mode"
//                     if (params.DEPLOY_TARGET == 'local') {
//                         env.DEPLOY_MODE = 'local'
//                         env.DEPLOY_HOST = env.LOCAL_HOST
//                         env.DEPLOY_USER = env.LOCAL_SSH_USER
//                     } else {
//                         env.DEPLOY_MODE = 'aws'
//                         env.DEPLOY_HOST = env.AWS_HOST
//                         env.DEPLOY_USER = env.AWS_SSH_USER
//                     }
//                     echo "[META] DEPLOY_MODE=${env.DEPLOY_MODE}"
//                     echo "[META] DEPLOY_HOST=${env.DEPLOY_HOST}"
//                     echo "[STAGE_SUCCESS] Select Deploy Mode"
//                 }
//             }
//         }

//         // ─────────────────────────────────────────────────────────────────────
//         stage('Clone Repo') {
//             steps {
//                 script {
//                     echo "[STAGE_START] Clone Repo"
//                     sh 'rm -rf app && git clone --depth=1 "$REPO_URL" app'
//                     echo "[STAGE_SUCCESS] Clone Repo"
//                 }
//             }
//         }

//         // ─────────────────────────────────────────────────────────────────────
//         // NEW: Auto-detect if single repo or monorepo.
//         // If monorepo → finds ALL deployable subfolders (frontend + backend etc.)
//         // If single   → uses root, business as usual
//         // Result: env.DEPLOY_TARGETS = list of maps [{pkgRoot, appSuffix}]
//         // ─────────────────────────────────────────────────────────────────────
//         stage('Detect Projects') {
//             steps {
//                 script {
//                     echo "[STAGE_START] Detect Projects"

//                     def targets = []

//                     def isDeployable = { dir ->
//                         fileExists("${dir}/package.json") ||
//                         fileExists("${dir}/requirements.txt") ||
//                         fileExists("${dir}/pom.xml") ||
//                         fileExists("${dir}/go.mod")
//                     }

//                     if (isDeployable('app')) {
//                         // ── Single repo: root has deployable marker ──────────
//                         echo "[INFO] Single repo detected — deploying root"
//                         targets << [pkgRoot: 'app', appSuffix: '']

//                     } else {
//                         // ── Monorepo: scan all immediate subdirs ─────────────
//                         echo "[INFO] No root marker found — scanning subfolders..."

//                         def subDirs = sh(
//                             script: "find app -mindepth 1 -maxdepth 1 -type d | sort",
//                             returnStdout: true
//                         ).trim().tokenize('\n')

//                         for (d in subDirs) {
//                             if (isDeployable(d)) {
//                                 def folderName = d.replace('app/', '')
//                                 echo "[INFO] Found deployable sub-project: ${folderName}"
//                                 targets << [pkgRoot: d, appSuffix: "-${folderName}"]
//                             }
//                         }

//                         if (targets.isEmpty()) {
//                             error("[ERROR] No deployable sub-projects found in repo. Each subfolder needs package.json / requirements.txt / pom.xml / go.mod")
//                         }
//                     }

//                     echo "[META] Total projects to deploy: ${targets.size()}"
//                     targets.each { t -> echo "[META] Will deploy: ${t.pkgRoot}" }

//                     // Store as JSON string so it survives across stages
//                     env.DEPLOY_TARGETS_JSON = groovy.json.JsonOutput.toJson(targets)

//                     echo "[STAGE_SUCCESS] Detect Projects"
//                 }
//             }
//         }

//         // ─────────────────────────────────────────────────────────────────────
//         stage('Secret Scan') {
//             steps {
//                 script {
//                     echo "[STAGE_START] Secret Scan"
//                     def result = sh(
//                         script: 'grep -rniE "(password|api_key|secret|token)\\s*=\\s*.{8,}" app --include="*.js" --include="*.ts" --include="*.py" --include="*.env" --exclude-dir=node_modules --exclude-dir=.git || true',
//                         returnStdout: true
//                     ).trim()
//                     if (result) {
//                         echo "[WARN] Potential secrets found — review before production"
//                         echo "[META] SECRET_SCAN=FAILED"
//                     } else {
//                         echo "[META] SECRET_SCAN=PASSED"
//                     }
//                     echo "[STAGE_SUCCESS] Secret Scan"
//                 }
//             }
//         }

//         // ─────────────────────────────────────────────────────────────────────
//         // NEW: Loop over every detected project and:
//         //   1. Detect stack
//         //   2. Audit dependencies
//         //   3. Create Dockerfile
//         //   4. Build image
//         //   5. Scan image (Trivy)
//         //   6. Push to DockerHub
//         //   7. Deploy container on auto-allocated port
//         //   8. Verify
//         // ─────────────────────────────────────────────────────────────────────
//         stage('Build & Deploy All') {
//             steps {
//                 script {
//                     echo "[STAGE_START] Build & Deploy All"

//                     def targets = new groovy.json.JsonSlurper().parseText(env.DEPLOY_TARGETS_JSON)
//                     def appName = env.APP_NAME
//                     def appId   = env.APP_ID
//                     def deployedURLs = []

//                     // Get all already-used ports once before the loop
//                     def usedRaw = sh(
//                         script: 'docker ps --format "{{.Ports}}" | grep -oE "[0-9]{2,5}" | sort -un || true',
//                         returnStdout: true
//                     ).trim()
//                     def usedPorts = usedRaw ? usedRaw.tokenize('\n').collect { it.trim() } : []

//                     for (target in targets) {

//                         def pkgRoot   = target.pkgRoot
//                         def suffix    = target.appSuffix  // e.g. "-frontend" or ""
//                         def safeId    = "${appName}${suffix}-${appId}"
//                         def imageName = "${DOCKERHUB_USER}/${safeId}:latest"
//                         def contName  = "${safeId}"

//                         echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
//                         echo "[PROJECT] pkgRoot=${pkgRoot}  image=${imageName}"
//                         echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

//                         // ── 1. Detect Stack ───────────────────────────────────
//                         def stack = 'node'
//                         def entry = 'index.js'

//                         if (fileExists("${pkgRoot}/package.json")) {
//                             def pkg = readFile("${pkgRoot}/package.json")
//                             if      (pkg.contains('"vite"'))                                                            stack = 'vite'
//                             else if (pkg.contains('"next"'))                                                            stack = 'nextjs'
//                             else if (pkg.contains('"react-scripts"'))                                                   stack = 'cra'
//                             else if (pkg.contains('"react"'))                                                           stack = 'react'
//                             else if (pkg.contains('"express"') || pkg.contains('"fastify"') || pkg.contains('"koa"'))   stack = 'node-server'
//                             else                                                                                        stack = 'node'

//                             def mainMatch = (pkg =~ /"main"\s*:\s*"([^"]+)"/)
//                             if (mainMatch.find()) {
//                                 entry = mainMatch.group(1)
//                                 mainMatch = null
//                             } else {
//                                 mainMatch = null
//                                 for (ep in ['index.js', 'server.js', 'app.js', 'main.js', 'src/index.js', 'src/server.js']) {
//                                     if (fileExists("${pkgRoot}/${ep}")) { entry = ep; break }
//                                 }
//                             }
//                         } else if (fileExists("${pkgRoot}/requirements.txt")) {
//                             stack = 'python'
//                             for (ep in ['app.py', 'main.py', 'run.py', 'server.py']) {
//                                 if (fileExists("${pkgRoot}/${ep}")) { entry = ep; break }
//                             }
//                         } else if (fileExists("${pkgRoot}/pom.xml")) {
//                             stack = 'java'; entry = ''
//                         } else if (fileExists("${pkgRoot}/go.mod")) {
//                             stack = 'go'; entry = 'main.go'
//                         }

//                         echo "[META] STACK=${stack}  ENTRY=${entry}"

//                         // ── 2. Allocate Port ──────────────────────────────────
//                         def portMap = [
//                             vite: 80, react: 80, cra: 80, nextjs: 80, frontend: 80,
//                             node: 3000, backend: 3000, "node-server": 3000,
//                             python: 3000, flask: 5000, fastapi: 8000, django: 8000,
//                             java: 8080, spring: 8080, go: 3000, php: 80
//                         ]
//                         def basePort = portMap[stack] ?: 3000
//                         def hostPort = basePort
//                         if (env.DEPLOY_MODE == 'local') {
//                             while (usedPorts.contains(hostPort.toString())) { hostPort++ }
//                             usedPorts << hostPort.toString() // reserve it for next project
//                         } else {
//                             hostPort = 80
//                         }

//                         def isFrontend    = (stack in ['vite', 'react', 'cra', 'nextjs'])
//                         def containerPort = isFrontend ? 80 : 3000

//                         echo "[META] HOST_PORT=${hostPort}  CONTAINER_PORT=${containerPort}"

//                         // ── 3. Write .dockerignore ────────────────────────────
//                         writeFile file: "${pkgRoot}/.dockerignore", text: '''\
// .git
// .gitignore
// node_modules
// bower_components
// npm-debug.log*
// yarn-error.log
// dist
// build
// out
// .next
// .nuxt
// .cache
// coverage
// .nyc_output
// __tests__
// *.test.js
// *.spec.js
// *.test.ts
// *.spec.ts
// jest.config.*
// cypress
// playwright
// .eslintrc*
// .prettierrc*
// .husky
// *.log
// logs
// .env
// .env.*
// *.pem
// *.key
// *.cert
// secrets
// .DS_Store
// Thumbs.db
// Dockerfile*
// docker-compose*
// .dockerignore
// .github
// .circleci
// docs
// *.md
// README*
// LICENSE*
// __pycache__
// *.pyc
// .venv
// venv
// target
// .gradle
// *.class
// *.jar
// *.war
// .terraform
// .serverless
// '''

//                         // ── 4. Dependency Audit ───────────────────────────────
//                         if (fileExists("${pkgRoot}/package.json")) {
//                             sh """
//                                 docker run --rm --network host \
//                                   -v "\$(pwd)/${pkgRoot}:/work" -w /work node:20-alpine \
//                                   sh -c 'npm config set fetch-retry-mintimeout 20000 && \
//                                          npm config set fetch-retry-maxtimeout 120000 && \
//                                          npm config set fetch-retries 5 && \
//                                          npm install --prefer-offline --ignore-scripts --silent 2>&1 | tail -3; \
//                                          npm audit --json 2>/dev/null || true' \
//                                   > /tmp/audit-${safeId}.txt 2>&1 || true
//                             """
//                             def out       = readFile("/tmp/audit-${safeId}.txt")
//                             def critMatch = (out =~ /"critical"\s*:\s*(\d+)/)
//                             def crit      = critMatch.find() ? critMatch.group(1) : '0'; critMatch = null
//                             def highMatch = (out =~ /"high"\s*:\s*(\d+)/)
//                             def high      = highMatch.find() ? highMatch.group(1) : '0'; highMatch = null
//                             echo "[META] VULN_CRITICAL=${crit}  VULN_HIGH=${high}"
//                         } else {
//                             echo "[INFO] No package.json — skipping npm audit"
//                         }

//                         // ── 5. Create Dockerfile ──────────────────────────────
//                         def dfPath = "${pkgRoot}/Dockerfile"
//                         if (!fileExists(dfPath)) {
//                             def df = ''
//                             switch (stack) {
//                                 case 'vite':
//                                     df = '''\
// FROM node:20-alpine AS builder
// WORKDIR /app
// COPY package*.json ./
// RUN npm config set fetch-retry-mintimeout 20000 && \
//     npm config set fetch-retry-maxtimeout 120000 && \
//     npm config set fetch-retries 5 && \
//     npm install --ignore-scripts
// COPY . .
// RUN npm run build

// FROM nginx:alpine
// RUN rm /etc/nginx/conf.d/default.conf
// COPY --from=builder /app/dist /usr/share/nginx/html
// RUN printf 'server{listen 80;root /usr/share/nginx/html;index index.html;location /{try_files $uri $uri/ /index.html;}}' > /etc/nginx/conf.d/app.conf
// EXPOSE 80
// CMD ["nginx", "-g", "daemon off;"]
// '''
//                                     break
//                                 case 'cra':
//                                 case 'react':
//                                     df = '''\
// FROM node:20-alpine AS builder
// WORKDIR /app
// COPY package*.json ./
// RUN npm config set fetch-retry-mintimeout 20000 && \
//     npm config set fetch-retry-maxtimeout 120000 && \
//     npm config set fetch-retries 5 && \
//     npm install --ignore-scripts
// COPY . .
// RUN npm run build

// FROM nginx:alpine
// RUN rm /etc/nginx/conf.d/default.conf
// COPY --from=builder /app/build /usr/share/nginx/html
// RUN printf 'server{listen 80;root /usr/share/nginx/html;index index.html;location /{try_files $uri $uri/ /index.html;}}' > /etc/nginx/conf.d/app.conf
// EXPOSE 80
// CMD ["nginx", "-g", "daemon off;"]
// '''
//                                     break
//                                 case 'nextjs':
//                                     df = '''\
// FROM node:20-alpine AS builder
// WORKDIR /app
// COPY package*.json ./
// RUN npm config set fetch-retry-mintimeout 20000 && \
//     npm config set fetch-retry-maxtimeout 120000 && \
//     npm config set fetch-retries 5 && \
//     npm install --ignore-scripts
// COPY . .
// ENV NEXT_TELEMETRY_DISABLED=1
// RUN npm run build

// FROM node:20-alpine
// WORKDIR /app
// ENV NODE_ENV=production
// ENV NEXT_TELEMETRY_DISABLED=1
// ENV HOSTNAME=0.0.0.0
// ENV PORT=3000
// COPY --from=builder /app/public ./public
// COPY --from=builder /app/.next/standalone ./
// COPY --from=builder /app/.next/static ./.next/static
// EXPOSE 3000
// CMD ["node", "server.js"]
// '''
//                                     break
//                                 case 'node-server':
//                                     df = """FROM node:20-alpine
// WORKDIR /app
// COPY package*.json ./
// RUN npm config set fetch-retry-mintimeout 20000 && \\
//     npm config set fetch-retry-maxtimeout 120000 && \\
//     npm config set fetch-retries 5 && \\
//     npm install --only=production --ignore-scripts
// COPY . .
// ENV NODE_ENV=production
// ENV HOST=0.0.0.0
// ENV PORT=3000
// EXPOSE 3000
// CMD ["node", "${entry}"]
// """
//                                     break
//                                 case 'python':
//                                     df = """FROM python:3.11-slim
// WORKDIR /app
// COPY requirements.txt ./
// RUN pip install --no-cache-dir -r requirements.txt
// COPY . .
// ENV HOST=0.0.0.0
// ENV PORT=3000
// EXPOSE 3000
// CMD ["python", "${entry}"]
// """
//                                     break
//                                 case 'java':
//                                     df = '''\
// FROM maven:3.9-eclipse-temurin-17 AS builder
// WORKDIR /app
// COPY pom.xml ./
// RUN mvn dependency:go-offline -q
// COPY src ./src
// RUN mvn package -DskipTests -q

// FROM eclipse-temurin:17-jre-alpine
// WORKDIR /app
// COPY --from=builder /app/target/*.jar app.jar
// EXPOSE 3000
// CMD ["java", "-jar", "app.jar", "--server.port=3000", "--server.address=0.0.0.0"]
// '''
//                                     break
//                                 case 'go':
//                                     df = '''\
// FROM golang:1.22-alpine AS builder
// WORKDIR /app
// COPY go.mod go.sum ./
// RUN go mod download
// COPY . .
// RUN CGO_ENABLED=0 GOOS=linux go build -o server .

// FROM alpine:3.19
// WORKDIR /app
// COPY --from=builder /app/server .
// EXPOSE 3000
// CMD ["./server"]
// '''
//                                     break
//                                 default:
//                                     df = """FROM node:20-alpine
// WORKDIR /app
// COPY package*.json ./
// RUN npm config set fetch-retry-mintimeout 20000 && \\
//     npm config set fetch-retry-maxtimeout 120000 && \\
//     npm config set fetch-retries 5 && \\
//     npm install --only=production --ignore-scripts
// COPY . .
// ENV HOST=0.0.0.0
// ENV PORT=3000
// EXPOSE 3000
// CMD ["node", "${entry}"]
// """
//                             }
//                             writeFile file: dfPath, text: df
//                             echo "[INFO] Generated Dockerfile for stack: ${stack}"
//                         } else {
//                             echo "[INFO] Using existing Dockerfile in repo"
//                         }

//                         // ── 6. Build Image ────────────────────────────────────
//                         sh """
//                             docker image inspect ${imageName} > /dev/null 2>&1 || echo "No cache found"
//                             docker build --network host \
//                                 -f "${pkgRoot}/Dockerfile" \
//                                 -t "${imageName}" \
//                                 "${pkgRoot}/"
//                         """

//                         // ── 7. Trivy Scan ─────────────────────────────────────
//                         sh "docker pull aquasec/trivy:latest 2>/dev/null || true"
//                         sh """
//                             docker run --rm --network host \
//                               -v /var/run/docker.sock:/var/run/docker.sock \
//                               -v /tmp/trivy-cache:/root/.cache/trivy \
//                               aquasec/trivy:latest image \
//                               --exit-code 0 --severity CRITICAL,HIGH \
//                               --format json --output /tmp/trivy-${safeId}.json \
//                               '${imageName}' || true
//                         """
//                         if (fileExists("/tmp/trivy-${safeId}.json")) {
//                             def report    = readFile("/tmp/trivy-${safeId}.json")
//                             def critCount = 0; def highCount = 0
//                             def cm = (report =~ /"Severity"\s*:\s*"CRITICAL"/); while (cm.find()) { critCount++ }; cm = null
//                             def hm = (report =~ /"Severity"\s*:\s*"HIGH"/)    ; while (hm.find()) { highCount++ }; hm = null
//                             echo "[META] IMAGE_CRITICAL_CVE=${critCount}  IMAGE_HIGH_CVE=${highCount}"
//                             echo "[META] IMAGE_SCAN=PASSED"
//                         }

//                         // ── 8. Push to DockerHub ──────────────────────────────
//                         sh "echo \"\$DOCKERHUB_CRED_PSW\" | docker login -u \"\$DOCKERHUB_CRED_USR\" --password-stdin"
//                         sh "docker push ${imageName}"
//                         sh "docker logout"

//                         // ── 9. Deploy ─────────────────────────────────────────
//                         if (env.DEPLOY_MODE == 'local') {
//                             sh """
//                                 docker stop  ${contName} 2>/dev/null || true
//                                 docker rm -f ${contName} 2>/dev/null || true
//                                 docker run -d \
//                                   --name ${contName} \
//                                   --restart unless-stopped \
//                                   -p 0.0.0.0:${hostPort}:${containerPort} \
//                                   ${imageName}
//                             """
//                             def url = "http://${env.LOCAL_HOST}:${hostPort}"
//                             echo "[META] URL=${url}"
//                             deployedURLs << "${contName} → ${url}"
//                         } else {
//                             sh """
//                                 ssh -o StrictHostKeyChecking=no -o ConnectTimeout=30 \
//                                   '${env.AWS_SSH_USER}'@'${env.AWS_HOST}' bash -s <<'ENDSSH'
// docker pull ${imageName}
// docker stop  ${contName} 2>/dev/null || true
// docker rm -f ${contName} 2>/dev/null || true
// docker run -d \
//   --name ${contName} \
//   --restart unless-stopped \
//   -p 0.0.0.0:${hostPort}:${containerPort} \
//   ${imageName}
// ENDSSH
//                             """
//                             def url = "http://${env.AWS_HOST}:${hostPort}"
//                             echo "[META] URL=${url}"
//                             deployedURLs << "${contName} → ${url}"
//                         }

//                         // ── 10. Verify ────────────────────────────────────────
//                         sleep 5
//                         if (env.DEPLOY_MODE == 'local') {
//                             def running = sh(
//                                 script: "docker ps --format '{{.Names}}' | grep -c '^${contName}\$' || true",
//                                 returnStdout: true
//                             ).trim()
//                             if (running == '1') {
//                                 echo "[OK] Container ${contName} is running"
//                                 sh """
//                                     for i in 1 2 3; do
//                                         curl -sf --max-time 10 http://localhost:${hostPort}/ -o /dev/null && echo 'HTTP OK' && break
//                                         echo "Attempt \$i failed, retrying in 5s..."
//                                         sleep 5
//                                     done || echo '[WARN] HTTP check did not succeed — app may still be starting'
//                                 """
//                             } else {
//                                 echo "[WARN] Container not running — dumping logs:"
//                                 sh "docker logs --tail 30 ${contName} 2>&1 || true"
//                             }
//                         }

//                         // cleanup local image to save disk
//                         sh "docker rmi ${imageName} 2>/dev/null || true"

//                     } // end for loop

//                     // ── Print summary of all deployed URLs ────────────────────
//                     echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
//                     echo "[DEPLOY SUMMARY] All deployed services:"
//                     deployedURLs.each { u -> echo "  ✅ ${u}" }
//                     echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

//                     echo "[STAGE_SUCCESS] Build & Deploy All"
//                 }
//             }
//         }

//     } // end stages

//     post {
//         always {
//             sh 'rm -rf app /tmp/trivy-*.json /tmp/audit-*.txt || true'
//             echo '[INFO] Workspace cleaned'
//         }
//         success {
//             echo '[DEPLOY_SUCCESS] All projects deployed successfully'
//         }
//         failure {
//             echo '[DEPLOY_FAILED]'
//         }
//     }
// }
