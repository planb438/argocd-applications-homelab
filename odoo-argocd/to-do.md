#### The initialization pod is stuck in "ContainerCreating" state. Let's check what's wrong and fix it:

#### Step 1: Check Why the Pod is Stuck
#### bash
# Check the pod status
    kubectl describe pod -n odoo odoo-init-2

# Check events
    kubectl get events -n odoo --sort-by='.lastTimestamp' | tail -20
#### Step 2: Use the Web Interface Method (Easiest)
#### Since the database is now empty and ready, let's use Odoo's web interface:

#### bash
# 1. Make sure Odoo is running
    kubectl scale deployment odoo -n odoo --replicas=1

# 2. Wait for Odoo to start
    sleep 10

# 3. Port forward
    kubectl port-forward -n odoo service/odoo-nodeport 8069:80

# 4. Open browser and go to:
# http://localhost:8069/web/database/manager
#### Step 3: If Web Interface Doesn't Work, Try Direct Init
#### bash
# 1. Delete the stuck pod
    kubectl delete pod -n odoo odoo-init-2

# 2. Scale Odoo to 0
    kubectl scale deployment odoo -n odoo --replicas=0

# 3. Verify the database is empty
    kubectl exec -n odoo postgres-0 -- psql -U odoo -d odoo -c "\dt"

# 4. Try initialization with a simpler command
    kubectl run -n odoo odoo-init-3 --image=odoo:17 --restart=Never -- \
      sh -c "odoo -d odoo -i base --db_host=postgres --db_user=odoo --db_password=odoo123 --without-demo=all --stop-after-init"

# 5. Watch the logs
    kubectl logs -n odoo odoo-init-3 -f

# 6. After it completes, scale Odoo
    kubectl scale deployment odoo -n odoo --replicas=1
#### Step 4: If Still Stuck, Use a Different Image or Approach
#### bash
# 1. Delete all init pods
    kubectl delete pod -n odoo -l run=odoo-init

# 2. Use a different approach - temporarily modify the deployment
    kubectl patch deployment odoo -n odoo --type='json' -p='[
      {
        "op": "add",
        "path": "/spec/template/spec/containers/0/args",
        "value": ["-i", "base", "--without-demo=all", "--stop-after-init"]
      }
    ]'

# 3. Scale Odoo to 1 (it will initialize and exit)
    kubectl scale deployment odoo -n odoo --replicas=1

# 4. Check logs
    kubectl logs -n odoo -l app=odoo -f

# 5. Once you see "init complete" or "stopping", remove the args
    kubectl patch deployment odoo -n odoo --type='json' -p='[
      {
        "op": "remove",
        "path": "/spec/template/spec/containers/0/args"
      }
    ]'

# 6. Restart Odoo
    kubectl rollout restart deployment odoo -n odoo
#### Quick Workaround: Use the Database Manager
#### Since your database is now empty, the simplest way is to use the web interface:

#### bash
# 1. Ensure Odoo is running
    kubectl get pods -n odoo

# 2. Port forward
    kubectl port-forward -n odoo service/odoo-nodeport 8069:80

# 3. In your browser, go to:
# http://localhost:8069/web/database/manager
#### At the database manager page:

#### Master Password: admin

#### Database Name: odoo

#### Email: admin@example.com

#### Password: admin123

#### Click "Create"

#### After the database is created, you'll be able to log in at http://localhost:8069

#### If None of the Above Works, Check Resource Limits
#### bash
# Check if you have enough resources
    kubectl describe node

# Check the init pod events
    kubectl describe pod -n odoo odoo-init-3

# Check PVC status
    kubectl get pvc -n odoo
#### Try the web interface method (Step 2) first - it's the easiest and most reliable way to get Odoo working!
