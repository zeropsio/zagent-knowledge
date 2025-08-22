# Platform Guide - Add Services

## You're Adding New Services to an Existing Project

This guide is for when you need to **expand an existing project** with new services (like adding a payment service, worker, or new database).

## Quick Decision Tree

```bash
# First, see what you already have
mcp__zerops__discovery($projectId)
```

### Need a new database/cache/queue?
→ Import semi-managed service first

### Need a new feature/microservice?
→ Import runtime pair (dev + stage)

### Both?
→ Semi-managed first, then runtime

## Adding Semi-Managed Services

### When You Need Them
- New database for isolated data
- Cache for performance
- Queue for async processing
- Object storage for files

### Import Pattern
```yaml
# Check available types first
mcp__zerops__get_service_types()

# Import what you need
mcp__zerops__import_services(yaml: """
services:
  - hostname: events
    type: valkey@7.2
    mode: NON_HA
  - hostname: queue
    type: nats@2.10
    mode: NON_HA
""")

# Wait for services to become ACTIVE using discovery polling
```

### Auto-Generated Variables
Each semi-managed service provides connection vars:
- `{hostname}_connectionString`
- `{hostname}_hostname`
- `{hostname}_port`
- `{hostname}_user`
- `{hostname}_password`

## Adding Runtime Services

### Import with Dev/Stage Pair
```yaml
mcp__zerops__import_services(yaml: """
services:
  - hostname: paymentdev
    type: nodejs@22
    startWithoutCode: true  # Critical for dev!
  - hostname: paymentstage
    type: nodejs@22
""")

# Wait for dev services: ACTIVE, stage services: READY_TO_DEPLOY
# Mount only dev services before creating files
```

**Why startWithoutCode?** Without it, dev service won't start until you deploy code. With it, you can SSH immediately.

### Universal Laws Apply Here

**Law 2: Hello-World Pipeline Validation**

Even for additional services, you MUST validate the pipeline first:

**Law 1: Dev Server Before Code**

After hello-world verification, start the dev server before writing real code.

**The requirements:**

1. **Create minimal working app** (after mounting the dev service)
   ```bash
   # Get template for your runtime
   mcp__zerops__knowledge_base("nodejs", "dev_stage_example")

   # Create hello-world that uses new dependencies (only after mounting)
   cat > /var/www/paymentdev/index.js << 'EOF'
   const PORT = process.env.PORT || 3000;
   require('http').createServer((req, res) => {
     res.writeHead(200);
     res.end(`Payment service connected to: ${process.env.db_hostname}`);
   }).listen(PORT, '0.0.0.0');
   EOF
   ```

2. **Create zerops.yml with proper setups**
   ```yaml
   zerops:
     - setup: dev
       build:
         base: nodejs@22
         buildCommands:
           - npm install
         deployFiles: ./  # Keep source for dev
       run:
         base: nodejs@22
         ports:
           - port: 3000
             httpSupport: true
         start: zsc noop

     - setup: prod
       build:
         base: nodejs@22
         buildCommands:
           - npm install
           - npm run build
         deployFiles: dist
       run:
         base: nodejs@22
         ports:
           - port: 3000
             httpSupport: true
         start: npm start
   ```

3. **Test both deployments**
   ```bash
   # Deploy to dev
   ssh paymentdev "zcli push --serviceId={PAYMENTDEV_ID} --setup=dev"

   # Deploy to stage
   ssh paymentdev "zcli push --serviceId={PAYMENTSTAGE_ID} --setup=prod"

   # Verify both work
   curl http://paymentdev:3000
   curl http://paymentstage:3000
   ```

## Connecting New Services to Existing Ones

### Environment Variable Cascade
When your new service needs to be accessed by existing services:

1. **Set variables on new service**
   ```bash
   mcp__zerops__set_service_env(paymentId, "stripe_key", "sk_...")
   ```

2. **Restart existing services that need access**
   ```bash
   # Now apidev can use ${paymentdev_stripe_key}
   response = mcp__zerops__restart_service(apidevId)
   # Monitor completion with processId
   ```

### Service Discovery
New services automatically expose:
- `{hostname}_hostname` to all services
- `{hostname}_{PORT_NAME}` for named ports
- Any env vars you set (with prefix)

## Common Patterns

### Adding Background Worker
```yaml
services:
  - hostname: workerdev
    type: python@3.11
    startWithoutCode: true
  - hostname: workerstage
    type: python@3.11
```

Then create worker that connects to existing services:
```python
# Uses existing database and cache
DB_URL = os.environ['db_connectionString']
CACHE_HOST = os.environ['cache_hostname']
```

### Adding Admin Panel
```yaml
services:
  - hostname: admindev
    type: nodejs@22
    startWithoutCode: true
  - hostname: adminstage
    type: nodejs@22
```

### Adding Specialized Database
```yaml
services:
  - hostname: analytics
    type: postgresql@17
    mode: NON_HA
```

## Integration Checklist

- [ ] Import semi-managed services first (if needed)
- [ ] Wait for import completion
- [ ] Import runtime pairs with startWithoutCode
- [ ] Create hello-world for new service
- [ ] Test dev deployment
- [ ] Test stage deployment
- [ ] Set any required env vars
- [ ] Restart dependent services
- [ ] Update existing services to use new service
- [ ] Test integration between services

## Quick Troubleshooting

### Can't SSH to new dev service
- Did you use `startWithoutCode: true`?
- Is import complete? Check processId

### Existing service can't see new service vars
- Did you restart the existing service?
- Is the variable name format correct? `{hostname}_{var}`

### Stage deployment fails
- Hello-world working on dev first?
- Correct setup name in zerops.yml?
- All dependencies available?
