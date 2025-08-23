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
ssh apidev "cd /var/www && npm install"
ssh apidev "cd /var/www && npm run dev"
ssh webdev "cd /var/www && npm run build"

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

**Rule 1: No Direct Service Variable References**
```yaml
# ❌ WRONG - Direct reference not supported
API_URL: ${apidev_VARIABLE}  # Won't work

# ✅ CORRECT - Must be mapped through envVariables
run:
  envVariables:
    API_URL: ${apidev_hostname}  # Mapped explicitly
```

**Rule 2: Stage Services MUST Use Public URLs**
```yaml
# ❌ WRONG - Internal hostname for stage frontend
build:
  envVariables:
    VITE_API_URL: http://apistage:3000  # Frontend can't reach!

# ✅ CORRECT - Public URL for stage
build:
  envVariables:
    VITE_API_URL: ${RUNTIME_apistage_zeropsSubdomain}  # Public access
```

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
**CRITICAL**: `zeropsSubdomain` already includes the protocol and full URL
```bash
# ✅ CORRECT
API_URL: ${apistage_zeropsSubdomain}

# ❌ WRONG - Double protocol
API_URL: https://${apistage_zeropsSubdomain}
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

```bash
# ✅ CORRECT - Dev deployment with all required flags
ssh apidev "zcli push --serviceId={EXACT_ID_FROM_DISCOVERY} --setup=dev --deploy-git-folder"
# PARAMETER: run_in_background: true

# ✅ CORRECT - Stage deployment (no --deploy-git-folder)
ssh apistage "zcli push --serviceId={EXACT_ID_FROM_DISCOVERY} --setup=stage"
# PARAMETER: run_in_background: true

# ❌ WRONG - Missing --serviceId
ssh apidev "zcli push --setup=dev"  # WILL FAIL

# ❌ WRONG - Missing --deploy-git-folder for dev
ssh apidev "zcli push --serviceId={ID} --setup=dev"  # Won't include source
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

This shared knowledge should be referenced in all agent prompts to ensure consistency while avoiding massive duplication.