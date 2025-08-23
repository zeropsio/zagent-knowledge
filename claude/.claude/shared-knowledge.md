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
**Required in these scenarios:**
- **NEW services with startWithoutCode** - Initial filesystem mounting required
- **Broken filesystem access** - "Transport endpoint not connected" errors
- **After network connectivity issues** - Refresh SSHFS connections
- **After service restarts** - Re-establish filesystem connections
- **After deploying new version** - Mount updated service filesystem

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

### CRITICAL Rule - No Hardcoded Hostnames:
**NEVER hardcode service hostnames in application code:**
```javascript
// ❌ WRONG - Hardcoded hostname in application
const API_URL = 'http://apidev:3000';  

// ✅ CORRECT - Use environment variable
const API_URL = process.env.API_URL || process.env.BACKEND_URL;
// Framework-specific patterns exist (see your framework's docs)
// Common pattern: PUBLIC_ or APP_ prefix for client-side variables
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

### Dev vs Stage Deployment Flags:
```bash
# DEV: Include source code and dependencies
ssh apidev "zcli push --serviceId={DEV_ID} --setup=dev --deploy-git-folder"

# STAGE: Built artifacts only
ssh apistage "zcli push --serviceId={STAGE_ID} --setup=stage"
```

### Service Type Patterns:
- **Frontend DEV**: `nodejs@22` (needs dev server)
- **Frontend STAGE**: `static` (CSR) or `nodejs@22` (SSR)
- **Backend**: Always `nodejs@22` (or appropriate runtime) for both
- **Databases**: `postgresql@17`, `valkey@7.2`, etc. (no dev/stage)

This shared knowledge should be referenced in all agent prompts to ensure consistency while avoiding massive duplication.