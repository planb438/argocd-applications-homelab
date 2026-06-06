This is a common WordPress PHP upload size limit issue. Let me help you fix it in your Kubernetes deployment.

---

Step 1: Check current PHP limits
First, let's verify the current limits:

bash
# Create a test PHP info pod
kubectl run php-info --rm -it --restart=Never --image=wordpress:latest -- bash -c "php -r \"echo 'upload_max_filesize: ' . ini_get('upload_max_filesize') . '\npost_max_size: ' . ini_get('post_max_size') . '\n';\""

---


Step 2: Fix using WordPress ConfigMap
Create a ConfigMap with custom PHP configuration:

bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-php-config
  namespace: wordpress
data:
  uploads.ini: |
    upload_max_filesize = 256M
    post_max_size = 256M
    max_execution_time = 300
    max_input_time = 300
    memory_limit = 256M
EOF

---


Step 3: Update WordPress Deployment
Update your WordPress deployment to mount the custom PHP config:

bash
kubectl edit deployment wordpress -n wordpress
Add this volume and volumeMount (or create a patch):

bash
cat <<EOF | kubectl patch deployment wordpress -n wordpress --type='strategic' -p '
spec:
  template:
    spec:
      containers:
      - name: wordpress
        volumeMounts:
        - name: php-config
          mountPath: /usr/local/etc/php/conf.d/uploads.ini
          subPath: uploads.ini
      volumes:
      - name: php-config
        configMap:
          name: wordpress-php-config
'
EOF


---


Step 4: Alternative - Create a Custom WordPress Docker Image
If the above doesn't work, create a custom Dockerfile:

dockerfile
FROM wordpress:latest

# Copy custom PHP configuration
RUN echo "upload_max_filesize = 64M" > /usr/local/etc/php/conf.d/uploads.ini && \
    echo "post_max_size = 64M" >> /usr/local/etc/php/conf.d/uploads.ini && \
    echo "max_execution_time = 300" >> /usr/local/etc/php/conf.d/uploads.ini && \
    echo "max_input_time = 300" >> /usr/local/etc/php/conf.d/uploads.ini && \
    echo "memory_limit = 256M" >> /usr/local/etc/php/conf.d/uploads.ini

# Also modify .htaccess for Nginx
RUN echo "php_value upload_max_filesize 64M" >> /var/www/html/.htaccess && \
    echo "php_value post_max_size 64M" >> /var/www/html/.htaccess && \
    echo "php_value max_execution_time 300" >> /var/www/html/.htaccess && \
    echo "php_value max_input_time 300" >> /var/www/html/.htaccess
Build and push:

bash
docker build -t yourregistry/wordpress-custom:latest .
docker push yourregistry/wordpress-custom:latest
Then update your deployment to use the custom image.


---


Step 5: Quick Fix Using WordPress Admin (If you can access)
Add this to your theme's functions.php or use a plugin:

php
// Add to functions.php
@ini_set( 'upload_max_size' , '64M' );
@ini_set( 'post_max_size', '64M');
@ini_set( 'max_execution_time', '300' );
Or use a plugin: Search for "Increase Maximum Upload File Size" in WordPress plugins.


---


Step 6: If Using Nginx Ingress
Also update your nginx ingress annotations:

bash
kubectl annotate ingress wordpress-ingress -n wordpress \
  nginx.ingress.kubernetes.io/proxy-body-size=64m \
  nginx.ingress.kubernetes.io/client-max-body-size=64m


---


Step 7: Verify the fix
After applying the fix, restart the WordPress pod:

bash
kubectl rollout restart deployment wordpress -n wordpress

# Wait for new pod
kubectl get pods -n wordpress -w

# Verify PHP limits
kubectl exec -n wordpress $(kubectl get pods -n wordpress -l app=wordpress -o name | head -1) -- php -r "echo 'upload_max_filesize: ' . ini_get('upload_max_filesize') . '\npost_max_size: ' . ini_get('post_max_size') . '\n';"
Complete Working Solution - Single YAML
Here's a complete patch that should work immediately:

bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-php-config
  namespace: wordpress
data:
  uploads.ini: |
    upload_max_filesize = 128M
    post_max_size = 128M
    max_execution_time = 600
    max_input_time = 600
    memory_limit = 512M
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  template:
    spec:
      containers:
      - name: wordpress
        volumeMounts:
        - name: php-config
          mountPath: /usr/local/etc/php/conf.d/uploads.ini
          subPath: uploads.ini
      volumes:
      - name: php-config
        configMap:
          name: wordpress-php-config
EOF
If you're using the Bitnami WordPress Helm chart
Update your values.yaml:

yaml
# values.yaml
php:
  uploadMaxFilesize: "128M"
  postMaxSize: "128M"
  maxExecutionTime: "600"
  memoryLimit: "512M"

resources:
  limits:
    memory: "1Gi"
    cpu: "500m"

ingress:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "128m"
Then upgrade:

bash
helm upgrade wordpress bitnami/wordpress -n wordpress -f values.yaml
