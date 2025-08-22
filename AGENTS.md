# Zerops Coding Agent

## 🚨 MANDATORY SESSION START

**CRITICAL: Do NOT create any TODOs until you complete the mandatory session and choose your path.**

**Phase 1 TODO - Create this ONLY:**
```
☐ Complete mandatory session (discovery + path selection)
☐ Load appropriate TODO template based on chosen path
```

**Mandatory Session Steps:**
```bash
echo $projectId  # MEMORIZE this value for all MCP calls
mcp__zerops__discovery($projectId)  # Get service IDs, hostnames, env vars
# 🚨🚨🚨 Choose Your Path at this point 🚨🚨🚨
```

## 🔴 TWO UNIVERSAL LAWS (ALL PATHS, ALL LANGUAGES)

### Law 1: Dev Server Runs BEFORE Code
**You cannot write code without a running dev server. Period.**
- The dev server is your feedback loop - without it you're blind
- Install dependencies first (npm install, pip install, go mod download, etc.)
- Start the development server (npm run dev, python app.py, go run ., etc.)
- Verify it responds (curl the endpoint)
- ONLY THEN write code

### Law 2: Hello-World Validates BEFORE Complexity
**For new services: Never build complex apps on untested pipelines.**
- Hello-world = minimal source + COMPLETE zerops.yml (with all real env vars)
- Test deployment to dev service
- Test deployment to stage service
- Both must succeed before writing real code
- This finds config issues with 10 lines, not 1000

## Choose Your Path & Load TODO Template

After Discovery, determine your goal and REPLACE your Phase 1 TODO with the appropriate template:

### Path A: Fresh Project → Full Setup Required
```bash
mcp__zerops__load_platform_guide("fresh_project")
# Use Path A TODO Template below
```

### Path B: Existing Service → Start Development
```bash
mcp__zerops__load_platform_guide("existing_service")
# Use Path B TODO Template below
```

### Path C: Add New Service(s) → Setup Then Develop
```bash
mcp__zerops__load_platform_guide("add_services")
# Use Path C TODO Template below
```

## TODO Templates (Replace Phase 1 TODO with chosen template)

### Path A TODO Template (Fresh Project)
```
☐ Import semi-managed services (databases, storage, etc.)
☐ Wait for ACTIVE status using discovery polling
☐ Import runtime services (dev/stage pairs)
☐ Wait for expected states and mount dev services
☐ Create hello-world with complete zerops.yml (Law 2)
☐ Test dev deployment with git init
☐ Test stage deployment
☐ Start dev server and verify responding (Law 1)
☐ [INSERT: Break down your specific application requirements here]
☐ Deploy to stage and verify
☐ Comprehensive testing
```

### Path B TODO Template (Existing Service)
```
☐ Start dev server and verify responding (Law 1)
☐ [INSERT: Break down your specific development tasks here]
☐ Test changes incrementally on dev server
☐ Deploy to stage when ready
☐ Verify stage deployment works
☐ Comprehensive testing
```

### Path C TODO Template (Add Services)
```
☐ Import new services (semi-managed first, then runtime)
☐ Wait for ACTIVE status using discovery polling  
☐ Mount dev services for new runtime services
☐ Create hello-world for new services with complete zerops.yml (Law 2)
☐ Test dev deployment with git init
☐ Test stage deployment
☐ Start dev server and verify responding (Law 1)
☐ [INSERT: Break down your specific application requirements here]
☐ Integrate with existing services
☐ Deploy to stage and verify
☐ Comprehensive testing
```

## Path B: Continuous Development (Most Common)

### Start Development (Law 1: Dev Server First)
```bash
# The pattern (adapt to your runtime):
# 1. Install dependencies (npm install, pip install, go mod download)
# 2. Start dev server (npm run dev, python app.py, go run ., etc.)
# 3. Verify responding (curl the endpoint)
# 4. ONLY THEN write code

# Example for Node.js:
ssh {service}dev "npm install && npm run dev"
curl http://{service}dev:3000

# NOW you can write code with immediate feedback
```

### Senior Developer Workflow (Law 1 Must Be Running)
**Prerequisite: Dev server is running and responding to requests**

1. **Implement incrementally**
   - Work on logical feature chunks
   - Don't write 500 lines blindly
   - Test changes immediately via dev server

2. **Test reasonably on dev**
   ```bash
   # APIs: Test endpoints as you build
   curl http://apidev:3000/api/users
   httpie POST apidev:3000/api/login user=test

   # Frontend: Check rendering and errors
   node .tools/puppeteer_check.js http://webdev:3000 \
     --check-console --check-network
   ```

3. **Deploy to stage when ready**
   ```bash
   # Review ENTIRE zerops.yml for needed changes
   # - New files in deployFiles?
   # - New build dependencies?
   # - New init commands?
   # - Environment variables?

   # CRITICAL: First deployment needs git init
   ssh {service}dev "git init && git add . && git commit -m 'Initial commit'"

   # Get setup name
   grep "setup:" {service}dev/zerops.yml

   # Deploy
   ssh {service}dev "zcli push --serviceId={STAGE_ID} --setup={SETUP_NAME}"
   ```

4. **Verify stage (MANDATORY)**
   ```bash
   curl http://{service}stage:3000/api/endpoint
   node .tools/puppeteer_check.js http://{service}stage:3000
   ```

## Command Routing Rules

### File Operations → Native on zagent
```bash
# Use your agent's tools directly on service files
# Agent operates in work directory with mounted services
```

### Runtime Commands → SSH to service
```bash
ssh {service} "npm install express"
ssh {service} "pip install requests"
ssh {service} "composer require laravel/framework"
```

### Service Communication → Direct HTTP
```bash
curl http://{service}:3000/endpoint
httpie GET {service}:8080/api/data
```

### Complex Operations → Pipe to zagent
```bash
ssh {service} "cat large.json" | jq '.data[]'
ssh {service} "find . -name '*.py'" | grep -v __pycache__
```

## Environment Variables (Major Failure Point!)

### Setting Variables - Understand the Impact

#### Project Level
```bash
mcp__zerops__set_project_env("API_KEY", "sk-1234...")
# Available to ALL services as API_KEY
# ⚠️ REQUIRES RESTART of all services using it!
```

#### Service Level
```bash
mcp__zerops__set_service_env(serviceId, "FEATURE_FLAG", "enabled")
# Available to others as {hostname}_FEATURE_FLAG
# ⚠️ REQUIRES RESTART of dependent services!
```

#### zerops.yml (Non-sensitive)
```yaml
run:
  envVariables:
    NODE_ENV: development
    DB_URL: ${db_connectionString}  # References auto-generated var
```
**⚠️ Only applied on DEPLOYMENT!**

### Correct Restart Pattern
```bash
# 1. Set variable
mcp__zerops__set_service_env(apiId, "NEW_FEATURE", "enabled")

# 2. Restart service (returns processId)
response = mcp__zerops__restart_service(workerdevId)
processId = response.processId

# 3. Monitor restart completion
mcp__zerops__get_running_processes(processId)
# Poll until process completes

# 4. Then verify var is available
ssh workerdev "echo \$apidev_NEW_FEATURE"
```

### Common Failures
- ❌ Expecting immediate availability after set
- ❌ Not restarting dependent services
- ❌ Wrong reference syntax
- ❌ Putting secrets in zerops.yml

## Deployment Excellence

### Pre-Deploy Review
```bash
# Review ENTIRE zerops.yml - not just deployFiles!
cat {service}dev/zerops.yml

# Check for needed updates:
# - deployFiles: includes new directories?
# - buildCommands: new dependencies?
# - initCommands: database migrations?
# - envVariables: new configs?
# - ports: new endpoints?

# Critical: deployFiles for dev MUST be ./
# Otherwise you'll lose source code!
```

### Deploy Command
```bash
ssh {service}dev "zcli push --serviceId={STAGE_ID} --setup={SETUP_NAME}"
# Streams logs until complete
```

### Post-Deploy
```bash
# 1. Verify stage
curl http://{service}stage:3000/api/status

# 2. Check mounts (disconnects on deploy/restart)
ls {service}dev/ || mcp__zerops__remount_service("{service}dev")
```

## Path A & C: New Service Setup

### The Hello-World Pattern (Law 2)

**Hello-World is NOT about simple code - it's about pipeline validation**

Components:
1. **Minimal source**: Just enough to verify service connectivity
2. **Complete zerops.yml**: Include ALL env vars your real app needs
3. **Full test cycle**: Deploy to dev, deploy to stage, verify both

**Why this prevents disasters:**
- Config errors surface with 10 lines of code, not 1000
- Missing env vars are caught before complex implementations
- Deployment issues are fixed before you've invested days in code
- You KNOW the pipeline works before building on it

### Setup Flow
```bash
# 1. Always start with databases/caches if needed
mcp__zerops__import_services(yaml: """
services:
  - hostname: db
    type: postgresql@17
  - hostname: cache
    type: valkey@7.2
""")

# 2. Wait for services to become ACTIVE using discovery polling

# 3. Import runtime pairs with startWithoutCode
mcp__zerops__import_services(yaml: """
services:
  - hostname: apidev
    type: nodejs@22
    startWithoutCode: true  # Starts empty, no deploy needed
  - hostname: apistage
    type: nodejs@22
""")

# Wait for dev services: ACTIVE, stage services: READY_TO_DEPLOY
# Mount only dev services before creating files

# Note: startWithoutCode allows immediate SSH access

# 4. Get appropriate template
mcp__zerops__knowledge_base("nodejs", "dev_stage_example")

# 5. Create hello-world, test dev→dev, dev→stage
# See platform guide for detailed steps
```

## Process Monitoring Pattern
```bash
# Operations return processId
response = mcp__zerops__scale_service(...)
processId = response.processId

# Poll for completion
status = mcp__zerops__get_running_processes(processId)
# Repeat until process disappears from list
```

## Import Services Pattern (CRITICAL)

**After ANY `mcp__zerops__import_services` call:**

1. **Wait for services to reach expected states** by calling `mcp__zerops__discovery(projectId)` every 10 seconds until:
   - Semi-managed services (databases, caches, storage): status = `ACTIVE`
   - Dev services with `startWithoutCode`: status = `ACTIVE`
   - Stage services without `startWithoutCode`: status = `READY_TO_DEPLOY`

2. **Mount dev services only** using `mcp__zerops__remount_service(hostname)` before file operations on that service

3. **Only then proceed** with hello-world creation or development work

**NEVER create files in service directories until dev services are ACTIVE and mounted. This will fail.**

## Preview URLs
```bash
# Get actual URL (exists before enabling)
ssh {service} "echo \$zeropsSubdomain"

# Enable public access
mcp__zerops__enable_preview_subdomain(serviceId)
```

## 🛑 BLOCKED ACTIONS

1. **Running on wrong service**
   ```bash
   npm install     # ❌ On zagent!
   ```

2. **Skipping dev server**
   ```bash
   # ❌ Editing without feedback loop
   ```

3. **Ignoring stage deployment**
   ```bash
   # ❌ "Dev works, ship it"
   ```

4. **Hardcoding anything**
   ```bash
   # ❌ --serviceId=abc123
   # ✅ Use Discovery data
   ```

5. **Creating configs from memory**
   ```bash
   # ❌ Writing zerops.yml manually
   # ✅ mcp__zerops__knowledge_base()
   ```

## Quick Patterns

### Multi-Service Development
```bash
# Start what you need
ssh apidev "npm run dev" &
ssh workerdev "python worker.py" &

# Work simultaneously
# Test integration points
curl http://apidev:3000/trigger-worker
```

### After Environment Changes
```bash
# Project env changed? Restart ALL services
# Service env changed? Restart dependents

response = mcp__zerops__restart_service(serviceId)
# Monitor with processId until complete
```

## Remember
- **Choose your path** after Discovery
- **Hello-world first** for new services
- **Dev server always** before coding
- **Restart after env changes** (and wait!)
- **Stage deployment mandatory**
- **Check mounts** after deploy/restart
