# Shared Knowledge for Zerops Agents

## 🚨 CRITICAL: Remote Service Architecture

**ALL operations happen on remote service containers, NOT locally:**

### File Operations vs Service Operations
```bash
# ✅ CORRECT - File editing uses mounted directories with native tools
Write("/var/www/apidev/package.json", content="...")
Edit("/var/www/webdev/src/main.js", old_string="...", new_string="...")
Read("/var/www/apidev/zerops.yml")

# ✅ CORRECT - Service operations use SSH to remote containers
ssh apidev "npm install"    # /var/www is default SSH directory
ssh apidev "npm run dev"    
ssh webdev "npm run build"

# ✅ CORRECT - Network testing targets remote services
curl http://apidev:3000/health
curl http://webdev:5173

# ❌ WRONG - Local operations (no runtime environment)
cd /var/www/apidev && npm run dev  # Would fail - no Node.js runtime locally
curl http://localhost:3000         # No server running locally
```

### Service Directory Mounts
```bash
# Service directories are ALREADY MOUNTED at startup:
# /var/www/apidev/   - mounted from apidev service
# /var/www/webdev/   - mounted from webdev service  
# /var/www/apistage/ - mounted from apistage service
```

### When to Use remount_service()
**ONLY use in these specific scenarios:**
- **NEW services with startWithoutCode** - Initial filesystem mounting required
- **After successful deployment to dev** - Mount updated filesystem (NOT stage)
- **Broken filesystem access** - "Transport endpoint not connected" errors
- **PM initial check** - If discovery shows ACTIVE dev but /var/www/{service}/ empty

**DO NOT use for:**
- Regular file operations
- Before every deployment
- As a "just in case" measure
- Stage services (they don't have mounts)
- When files are already accessible

```bash
# Example usage
mcp__zerops__remount_service("apidev")  # Returns mount command
# Execute the returned bash command to establish mount
```

## Background Process Management Protocol

**MANDATORY: ALL long-running operations MUST use `run_in_background: true`**

### Operations Requiring Background Execution:
- **Dev servers**: `npm run dev`, `python app.py`, `go run .`, `rails server`
- **Deployments**: `zcli push --serviceId=X --setup=dev`
- **Dependency installs**: `npm install`, `pip install -r requirements.txt`
- **Build processes**: `npm run build`, `tsc`, `webpack`
- **Database operations**: `npm run migrate`, `python manage.py migrate`
- **Any command taking >5 seconds or running continuously**

### Correct Usage Pattern:
```bash
# ✅ CORRECT - Allows parallel operations and monitoring
ssh apidev "npm run dev"
# PARAMETER: run_in_background: true
# Returns: {"bash_id": "abc123", "output": "..."}

# Monitor until ready
BashOutput(bash_id="abc123")  # Check until "Server running on port 3000"

# ❌ WRONG - Blocks all other operations  
ssh apidev "npm run dev"  # Without run_in_background

# ❌ WRONG - Shell background (unreliable)
ssh apidev "npm run dev &"
```

### Process Monitoring Workflow:
1. Start background command → Get bash_id
2. Use BashOutput(bash_id) to monitor progress
3. Wait for completion/ready message before proceeding
4. Only then test connectivity or continue workflow

### Monitoring Patterns with Error Handling:
```bash
# Starting a dev server
ssh apidev "npm run dev"
# PARAMETER: run_in_background: true
# Returns: {"bash_id": "dev_abc123"}

# Monitor for successful startup
BashOutput(bash_id="dev_abc123")
# Look for: "Server running on port 3000" or "Listening on"
# If output shows: "Error:" or "Failed" → Stop and diagnose

# Monitoring deployment
ssh apidev "zcli push --serviceId=abc123 --setup=dev --deploy-git-folder"
# PARAMETER: run_in_background: true
# Returns: {"bash_id": "deploy_xyz789"}

# Check deployment progress
BashOutput(bash_id="deploy_xyz789")
# Look for: "Successfully deployed" or "Deployment complete"
# If output shows: "Build failed" or hangs >60s → Check discovery for status

# Handling common scenarios:
# 1. If server crashes: Output will show error, restart with new bash_id
# 2. If deployment hangs: After 60s, check discovery() for service status
# 3. If no output: Process may have died, check with ps aux via SSH
# 4. If continuous errors: Escalate to operations-engineer

# Deployment verification pattern:
# Monitor output for success indicators
BashOutput(bash_id="deploy_xyz789")
# Look for: "Build successful", "Deployment complete", "Successfully deployed"
# If output stops but no success message after 60s:
mcp__zerops__discovery($projectId)  # Check if service status is ACTIVE
# If ACTIVE but no output: deployment succeeded in background
# If still DEPLOYING: check build logs
mcp__zerops__get_service_logs(serviceId, show_build_logs=true)
```

### CRITICAL: Separate Commands for Dependencies and Dev Server

```bash
# ❌ WRONG - Combined commands block proper monitoring
ssh apidev "npm install && npm run dev"

# ✅ CORRECT - Separate commands with proper monitoring
ssh apidev "npm install"
# Wait for completion
ssh apidev "npm run dev"
# PARAMETER: run_in_background: true
BashOutput(bash_id="dev_server")  # Monitor startup
```

## Environment Variable System

### The Three-Level Cascade:

**Level 1: Project Variables (Global)**
```bash
mcp__zerops__set_project_env("API_KEY", "sk-1234...")
# Available to ALL services as: ${API_KEY}
# ⚠️ REQUIRES RESTART of every service using them
```

**Level 2: Service Variables (Cross-Service)**
```bash
mcp__zerops__set_service_env(paymentdevId, "STRIPE_KEY", "sk_test...")
# Available to OTHER services as: ${paymentdev_STRIPE_KEY}
# ⚠️ CONSUMER services must restart to see them
```

**Level 3: zerops.yml Variables (Deploy-Time)**
```yaml
run:
  envVariables:
    NODE_ENV: production
    DB_URL: ${db_connectionString}        # Auto-generated
    API_KEY: ${API_KEY}                   # Project level
    PAYMENT: ${paymentdev_STRIPE_KEY}     # Service level
```

**⚠️ CRITICAL DISTINCTION:**
- **MCP env changes (Levels 1 & 2)**: Need RESTART only
- **zerops.yml changes (Level 3)**: Need DEPLOYMENT to take effect
- **Sensitive secrets**: Use MCP (avoid redeployments)
- **Build-time vars**: Must be in zerops.yml

**🚨 STATIC vs SSR Environment Variables:**
- **Static sites (CSR)**: Variables go in `build.envVariables` - NO RUNTIME_ prefix!
  - They're baked into the JS bundle at build time
  - Example: `VITE_API_URL: ${apistage_zeropsSubdomain}`
- **SSR sites**: Variables can go in `run.envVariables`
  - They're available at runtime on the server
  - Use RUNTIME_ prefix only when build needs runtime vars

### Critical Restart Pattern:
**When service A needs a new variable from service B:**
1. Set variable on service B (source)
2. **Restart service A (consumer)** - NOT service B!
3. Monitor restart completion using processId
4. Variable now available to service A

## Service Communication Standards

### Hostname Usage:
```bash
# ✅ Service-to-service (internal) - in zerops.yml
API_URL: http://${apidev_hostname}:3000

# ✅ External access via zeropsSubdomain  
PUBLIC_API: ${apistage_zeropsSubdomain}  # Already includes https://
```

### CRITICAL Environment Variable Rules

**Rule 1: Environment Variables Must Be Explicitly Mapped**
```yaml
# ❌ WRONG - Direct reference not supported in app code
# Your app CANNOT access ${apidev_hostname} directly

# ✅ CORRECT - Map Zerops variables to app variables in zerops.yml
run:
  envVariables:
    # Left side: What your app code expects (process.env.API_HOST)
    # Right side: Zerops system variable
    API_HOST: ${apidev_hostname}
    DATABASE_URL: ${db_connectionString}
    REDIS_URL: ${cache_hostname}
```

**Rule 2: The Universal Browser Rule**

**🔑 CORE PRINCIPLE: Browsers cannot reach internal service hostnames.**

Any URL that will be called from:
- A browser (Chrome, Firefox, Safari, etc.)
- A mobile app
- Any external client
- Postman/Insomnia on your local machine

...MUST use the public `zeropsSubdomain` URL, not internal hostnames.

```yaml
# ❌ WRONG - Browser can't reach internal hostname
API_URL: http://apistage:3000  # Fails from any browser!

# ✅ CORRECT - Use public URL for anything browser-accessible
API_URL: ${apistage_zeropsSubdomain}  # Works from anywhere

# The variable name (API_URL, BACKEND_URL, etc.) and where you put it 
# (build.envVariables or run.envVariables) depends on your framework.
# The principle remains: browsers need public URLs.
```

**Why this matters:**
- Internal hostnames (apidev, apistage) only work INSIDE the Zerops network
- Browsers run on users' machines OUTSIDE the network
- Public URLs (zeropsSubdomain) are accessible from anywhere
- This applies to ALL frameworks, ALL architectures, ALL apps

**Rule 3: Never Hardcode in Application Code**
```javascript
// ❌ WRONG - Hardcoded hostname
const API_URL = 'http://apidev:3000';

// ✅ CORRECT - Use environment variable
const API_URL = process.env.API_URL || process.env.VITE_API_URL;
```

**Testing from zagent is different** - you CAN use direct hostnames:
```bash
# ✅ OK for testing from zagent
curl http://apidev:3000/health  

# ✅ OK in SSH commands  
ssh apidev "curl http://localhost:3000/health"
```

### zeropsSubdomain Protocol:
**CRITICAL**: `zeropsSubdomain` is a COMPLETE URL including protocol
```bash
# zeropsSubdomain value example: "https://api-abc123.app.zerops.io"

# ✅ CORRECT - Use as-is
API_URL: ${apistage_zeropsSubdomain}
# Results in: API_URL="https://api-abc123.app.zerops.io"

# ❌ WRONG - Adding protocol creates invalid URL
API_URL: https://${apistage_zeropsSubdomain}
# Results in: API_URL="https://https://api-abc123.app.zerops.io"
```

## Agent Handoff Protocol

### Explicit Handoff Structure
When transferring work between agents, use this format:
```yaml
HANDOFF:
  from: infrastructure-architect
  to: main-developer
  state:
    services_created: [apidev, webdev, db]
    validation_status: hello-world passed
    env_vars_configured: [DB connection, API endpoints]
    dev_servers: killed and cleaned up
  next_steps: Feature implementation ready
  warnings: None
```

### Retry Limits and Human Escalation
- **Max 3 attempts** for any operation before escalating
- **Clear error message** after max attempts
- **Human intervention request** format:
  ```
  "Failed after 3 attempts. Human intervention needed:
  Issue: [Specific problem]
  Attempted: [What was tried]
  Suspected cause: [Best guess]
  Suggested action: [What human should check]"
  ```

## Agent Escalation Boundaries

### Infrastructure-Architect Handles:
- Service creation and import
- Architecture decisions  
- YAML configuration patterns
- Hello-world validation

### Operations-Engineer Handles:
- Environment variable issues
- Deployment troubleshooting
- System diagnostics
- Complex pipeline problems

### Main-Developer Handles:
- Feature implementation
- Bug fixes
- Simple deployments
- Code development

### Escalation Format:
```
"@[agent-name]: [SPECIFIC REQUIREMENT]

Context: [Current situation]
Problem: [Exact issue]  
Expected: [Desired outcome]"
```

## Common Deployment Patterns

### CRITICAL Deployment Command Structure

**MANDATORY: Always get serviceId from discovery, NEVER guess or omit**

### Extracting Service IDs from Discovery
```bash
# 1. Call discovery to get all service information
mcp__zerops__discovery($projectId)

# Discovery returns structure like:
# {
#   "services": [
#     {"id": "abc123-def456", "hostname": "apidev", "status": "ACTIVE", ...},
#     {"id": "ghi789-jkl012", "hostname": "apistage", "status": "ACTIVE", ...},
#     {"id": "mno345-pqr678", "hostname": "webdev", "status": "ACTIVE", ...}
#   ]
# }

# 2. Extract the specific service ID you need
# Look through the discovery response for the service with matching hostname
# In the response, find: {"id": "abc123-def456", "hostname": "apidev", ...}
# Extract that ID value: abc123-def456

# 3. Use the EXACT ID in deployment commands
ssh apidev "zcli push --serviceId=abc123-def456 --setup=dev --deploy-git-folder"
# PARAMETER: run_in_background: true

ssh apistage "zcli push --serviceId=ghi789-jkl012 --setup=stage"
# PARAMETER: run_in_background: true

# Example of extracting in practice:
# From discovery, if you see:
# "services": [{"id": "srv-abc123", "hostname": "apidev", "status": "ACTIVE"}]
# Then use: --serviceId=srv-abc123
```

### Deployment Command Examples
```bash
# ✅ CORRECT - Dev deployment with all required flags
ssh apidev "zcli push --serviceId=abc123-def456 --setup=dev --deploy-git-folder"
# PARAMETER: run_in_background: true

# ✅ CORRECT - Stage deployment (no --deploy-git-folder)
ssh apistage "zcli push --serviceId=ghi789-jkl012 --setup=stage"
# PARAMETER: run_in_background: true

# ❌ WRONG - Missing --serviceId
ssh apidev "zcli push --setup=dev"  # WILL FAIL

# ❌ WRONG - Missing --deploy-git-folder for dev
ssh apidev "zcli push --serviceId=abc123-def456 --setup=dev"  # Won't include source
```

### deployFiles Pattern - CRITICAL

```yaml
# ❌ WRONG - Creates /var/www/dist/index.html (404!)
deployFiles: dist

# ✅ CORRECT - Extracts contents to /var/www/index.html
deployFiles: dist/~  # The ~ wildcard is MANDATORY for static sites
```

### Service Type Patterns:
- **Frontend DEV**: `nodejs@22` (needs dev server)
- **Frontend STAGE**: `static` (CSR) or `nodejs@22` (SSR)
- **Backend**: Always `nodejs@22` (or appropriate runtime) for both
- **Databases**: `postgresql@17`, `valkey@7.2`, etc. (no dev/stage)

## Standard Handoff Protocol

**ALL agents MUST use this format when reporting completion:**
```
HANDOFF REPORT:
Agent: [your-name]
Status: SUCCESS/FAILED/PARTIAL
Completed:
  - [specific action with evidence]
  - [specific action with evidence]
Issues: [any problems encountered]
Next: [recommended agent or "PM decision"]
Evidence:
  - Services created: [list with IDs]
  - Tests passed: [URLs tested]
  - Code written: [files modified]
```

## Recovery Protocol (ALL AGENTS)

**When lost or taking over:**
```bash
# 1. Check project state
Read(".zmanager/state.md")  # PM-maintained state

# 2. Get actual state
mcp__zerops__discovery($projectId)

# 3. Check implementation
ls /var/www/{relevant-services}/
git log --oneline -5  # Recent commits

# 4. Check processes
ps aux | grep -E '(node|python|go)'

# 5. Resume from last known state
```

## Error Handling Patterns

### MCP Call Failures
```bash
# If knowledge_base fails
Retry once → Use cached patterns → Escalate to PM

# If import_services fails  
Check YAML syntax → Retry with corrections → Escalate to PM

# If restart/deploy fails
Check service status → Check logs → Escalate to operations-engineer
```

### Git Failures
```bash
# If git init fails
ssh service "rm -rf .git && git init"

# If git commit fails
ssh service "git add -A && git commit -m 'Force commit' --allow-empty"
```

## Error Boundaries and Recovery

### Critical Error Boundaries
**When to stop and escalate to human:**
1. **After 3 failed attempts** at any operation
2. **MCP functions consistently failing** (network/auth issues)
3. **Service creation failing repeatedly** (quota/permission issues)
4. **Deployment hanging >5 minutes** without progress
5. **Critical services down** and restart fails

### Error Response Patterns

#### Knowledge Base Failures
```bash
# First attempt fails
mcp__zerops__knowledge_base("nodejs")  # Returns error

# Recovery attempt 1: Try with different query
mcp__zerops__knowledge_base("node")

# Recovery attempt 2: Use known good patterns
# Fall back to basic validated configs

# After 3 failures: Escalate to human
"Failed to retrieve knowledge base patterns after 3 attempts.
Human intervention needed:
- Check MCP connection
- Verify knowledge base availability
- Consider using manual YAML configuration"
```

#### Service Import Failures
```bash
# Import fails with YAML error
mcp__zerops__import_services(yaml="...")  # Returns: "Invalid YAML"

# Recovery: Validate and fix YAML syntax
# Check for: indentation, missing colons, unclosed quotes

# Import fails with service conflict
# Recovery: Check existing services with discovery()
# Modify hostnames if conflicts found

# After 3 failures: Request human help
"Service import failed repeatedly. Please check:
- Project quotas and limits
- Service naming conflicts
- YAML syntax validation"
```

#### Deployment Failures
```bash
# Deployment hangs
ssh apidev "zcli push --serviceId=abc123 --setup=dev"
# No response after 60 seconds

# Recovery 1: Check if deployment actually succeeded
mcp__zerops__discovery($projectId)  # Check service status

# Recovery 2: Check deployment logs
mcp__zerops__get_service_logs(serviceId, show_build_logs=true)

# Recovery 3: Kill hung process and retry
ssh apidev "pkill -f zcli"
ssh apidev "zcli push --serviceId=abc123 --setup=dev --deploy-git-folder"

# After 3 attempts: Escalate
"Deployment consistently failing. Human should check:
- Service logs for build errors
- Git repository state
- Zerops.yml configuration"
```

### Graceful Degradation Patterns
**When non-critical operations fail:**
- Continue with core functionality
- Log issues for later resolution
- Inform user of degraded state
- Suggest manual workarounds

### Human Escalation Format
```
"⚠️ Human intervention required

Operation: [What was being attempted]
Failure: [Specific error or symptom]
Attempts: 3 (with variations)
Suspected cause: [Best guess based on symptoms]

Recommended actions:
1. [Specific check or fix]
2. [Alternative approach]
3. [Manual workaround if available]

Context preserved in .zmanager/error-state.md"
```

This shared knowledge should be referenced in all agent prompts to ensure consistency while avoiding massive duplication.