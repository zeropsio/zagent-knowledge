# Platform Guide - Existing Service

## You're Working on an Existing Service

This guide is for the **most common scenario** - the service already exists, hello-world was done, and you just need to develop.

## Law 1: Dev Server MUST Run First

**Universal requirement: You cannot code without a running dev server**

```bash
# 1. Install dependencies then start dev server (MANDATORY FIRST)
ssh {service}dev "npm install && npm run dev"     # Node.js
ssh {service}dev "pip install -r requirements.txt && python app.py"   # Python
ssh {service}dev "go mod download && go run ."        # Go
ssh {service}dev "bundle install && bundle exec rails server -b 0.0.0.0"  # Ruby
# PHP: composer install (starts automatically)

# 2. Verify it's responding (CRITICAL)
curl http://{service}dev:3000
curl http://{service}dev:3000/api/status

# 3. NOW you can code with immediate feedback
# Edit files directly in service directories
```

## Development Flow

### Code → Test → Deploy → Verify

1. **Make changes incrementally**
   - Edit files using your tools
   - Work on one feature at a time
   - Don't accumulate 20 uncommitted changes

2. **Test on dev server frequently**
   ```bash
   # API endpoints
   curl http://apidev:3000/api/new-endpoint
   httpie POST apidev:3000/api/auth username=test

   # Frontend
   node .tools/puppeteer_check.js http://webdev:3000
   ```

3. **When feature is ready, deploy to stage**
   ```bash
   # Check setup name
   grep "setup:" {service}dev/zerops.yml

   # Deploy
   ssh {service}dev "zcli push --serviceId={STAGE_ID} --setup={SETUP_NAME}"
   ```

4. **Verify on stage (MANDATORY)**
   ```bash
   curl http://{service}stage:3000/api/endpoint
   ```

## Common Development Tasks

### Adding Dependencies
```bash
# Node.js
ssh apidev "npm install express cors"

# Python
ssh workerdev "pip install requests redis"

# Go
ssh apidev "go get github.com/gin-gonic/gin"
```

### Checking Logs
```bash
# Dev server output is already streaming
# For historical logs:
mcp__zerops__get_service_logs({service}devId)
```

### Using Environment Variables
```bash
# Check what's available using discovery mcp call

# Common variables:
# - db_connectionString (from database)
# - cache_hostname (from cache service)
# - PROJECT_VARS (from project level)
# - {service}_{var} (from other services)
```

## Things That Require Restarts

### After Setting New Env Vars
```bash
# Set variable
mcp__zerops__set_service_env(serviceId, "FEATURE", "enabled")

# Restart and wait
response = mcp__zerops__restart_service({service}devId)
mcp__zerops__get_running_processes(response.processId)  # Poll

# Now available
ssh {service}dev "echo \$FEATURE"
```

### After Another Service's Env Changes
If service B sets a new env var and your service uses it:
```bash
# Must restart your service too
response = mcp__zerops__restart_service(yourServiceId)
```

## Quick Troubleshooting

### Dev Server Crashed
```bash
# Just restart it
ssh {service}dev "npm run dev"
```

### Can't See Files
```bash
# Mounts disconnected
ls {service}dev/ || mcp__zerops__remount_service("{service}dev")
```

### Stage Deploy Failed
- Check zcli output for errors
- Verify setup name exists in zerops.yml
- Check if deployFiles is correct

## Remember
- **Always run dev server first**
- **Test incrementally**
- **Deploy to stage when ready**
- **Verify stage works**
