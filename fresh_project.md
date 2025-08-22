# Platform Guide - Fresh Project

## You're Starting from Scratch

This guide is for **completely new projects** where you need to set up everything from the beginning.

## Universal Laws Apply Here

**Law 2: Hello-World Pipeline Validation (NON-NEGOTIABLE)**

Most deployment failures happen because people skip this step and go straight to complex code. Hello-world ensures your entire pipeline works BEFORE you write real code.

**Law 1: Dev Server Before Code**

After hello-world verification, you MUST start the dev server before writing your real application.

## Step-by-Step Fresh Project Setup

### 1. ALWAYS Start with Semi-Managed Services

```yaml
# Databases, caches, queues go FIRST
mcp__zerops__import_services(yaml: """
services:
  - hostname: db
    type: postgresql@17
    mode: NON_HA
  - hostname: cache
    type: valkey@7.2
    mode: NON_HA
  - hostname: storage
    type: object-storage
    objectStorageSize: 1
    objectStoragePolicy: public-read
""")

# Wait for services to become ACTIVE using discovery polling
```

**Available service types:** Use `mcp__zerops__get_service_types()` to see current list.

### 2. Import Runtime Service Pairs

```yaml
mcp__zerops__import_services(yaml: """
services:
  - hostname: apidev
    type: nodejs@22
    startWithoutCode: true  # KEY: Allows immediate SSH
  - hostname: apistage
    type: nodejs@22
  - hostname: workerdev
    type: python@3.11
    startWithoutCode: true
  - hostname: workerstage
    type: python@3.11
""")

# Wait for dev services: ACTIVE, stage services: READY_TO_DEPLOY
# Mount only dev services before creating files
```

**Why startWithoutCode?** Creates empty container you can SSH into immediately. Without this, container won't start until first deployment.

### 3. Wait for Services and Mount Before File Operations

**CRITICAL: Wait for dev services to be ACTIVE and mount them first**

#### For Node.js Service:
```bash
# Get template
mcp__zerops__knowledge_base("nodejs", "dev_stage_example")

# Create files (only after mounting)
cat > apidev/package.json << 'EOF'
{
  "name": "hello-api",
  "scripts": {
    "dev": "node index.js",
    "start": "node index.js"
  }
}
EOF

cat > apidev/index.js << 'EOF'
const PORT = process.env.PORT || 3000;
require('http').createServer((req, res) => {
  res.writeHead(200);
  res.end(`Hello from ${process.env.hostname}! DB: ${process.env.db_hostname}`);
}).listen(PORT, '0.0.0.0');
console.log(`Server running on port ${PORT}`);
EOF
```

#### Create zerops.yml:
```yaml
zerops:
  - setup: dev
    build:
      base: nodejs@22
      buildCommands:
        - npm install
      deployFiles: ./  # CRITICAL: Preserves source code
    run:
      base: nodejs@22
      ports:
        - port: 3000
          httpSupport: true
      start: zsc noop  # Empty for manual control
      envVariables:
        NODE_ENV: development

  - setup: prod
    build:
      base: nodejs@22
      buildCommands:
        - npm install
        - npm run build  # When you have build
      deployFiles: dist  # Only production files
    run:
      base: nodejs@22
      ports:
        - port: 3000
          httpSupport: true
      start: npm start
      envVariables:
        NODE_ENV: production
```

### 4. Test BOTH Deployments

```bash
# CRITICAL: Initialize git first
ssh apidev "git init && git add . && git commit -m 'Hello world setup'"

# Deploy to dev (tests pipeline, applies envs)
ssh apidev "zcli push --serviceId={APIDEV_ID} --setup=dev"

# Verify dev updated
curl http://apidev:3000

# Deploy to stage
ssh apidev "zcli push --serviceId={APISTAGE_ID} --setup=prod"

# Verify stage works
curl http://apistage:3000
```

**Only proceed to real development after BOTH work!**

### 5. Repeat for Other Services

Do the same hello-world setup for worker, web, etc. Each needs its own verified pipeline.

## Environment Variables for Fresh Projects

### Auto-Generated from Semi-Managed Services
```bash
# PostgreSQL provides:
db_connectionString
db_hostname
db_port
db_user
db_password

# Don't set these manually!
```

### Set Project-Wide Secrets First
```bash
mcp__zerops__set_project_env("API_KEY", "sk-...")
mcp__zerops__set_project_env("JWT_SECRET", "...")

# Restart all services to apply
response = mcp__zerops__restart_service(apidevId)
# Monitor with processId
```

### Reference in zerops.yml
```yaml
run:
  envVariables:
    DATABASE_URL: ${db_connectionString}
    API_KEY: ${API_KEY}  # From project env
```

## Common Fresh Project Mistakes

1. **Starting with runtime services** → Always databases first
2. **Skipping hello-world** → Guaranteed deployment failures later
3. **Not testing stage** → Production surprises
4. **Forgetting startWithoutCode** → Can't SSH to create files
5. **Wrong deployFiles in dev** → Code loss on deployment

## Fresh Project Checklist

- [ ] Import all semi-managed services
- [ ] Wait for completion (monitor processId)
- [ ] Import runtime pairs with startWithoutCode
- [ ] Create hello-world for each service
- [ ] Test dev deployment for each
- [ ] Test stage deployment for each
- [ ] Set all environment variables
- [ ] Restart services to apply envs
- [ ] Start dev server (Law 1)
- [ ] NOW start real development
