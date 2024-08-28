Certainly! Here’s the updated YAML configuration and explanation in Markdown format.

---

## Updated YAML Configuration for Apache with SSL

### Custom Initialization Script

Create a script named `setup-ssl.sh` to enable SSL and configure Apache within the container.

**`setup-ssl.sh`**

```bash
#!/bin/bash

# Enable SSL module and rewrite module
a2enmod ssl
a2enmod rewrite

# Create or modify the SSL configuration for your site
cat <<EOF > /etc/apache2/sites-available/wordpress-ssl.conf
<VirtualHost *:443>
    ServerName your-domain.com
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/tls.crt
    SSLCertificateKeyFile /etc/ssl/certs/tls.key

    <Directory /var/www/html>
        AllowOverride All
    </Directory>
</VirtualHost>
EOF

# Enable the SSL site configuration
a2ensite wordpress-ssl.conf

# Restart Apache to apply the changes
service apache2 restart
```

Ensure this script is executable:

```bash
chmod +x setup-ssl.sh
```

### Updated Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:apache # Assuming Apache-based WordPress image
          ports:
            - containerPort: 443 # SSL Port
          volumeMounts:
            - name: ca-certificates
              mountPath: /usr/local/share/ca-certificates/
            - name: ssl-certificates
              mountPath: /etc/ssl/certs/
          env:
            - name: WORDPRESS_HTTP_PORT
              value: "443"
            - name: WORDPRESS_HTTPS_PORT
              value: "443"
            - name: WORDPRESS_CONFIG_EXTRA
              value: |
                define('FORCE_SSL_ADMIN', true);
                if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false) {
                  $_SERVER['HTTPS'] = 'on';
                }
      initContainers:
        - name: setup-apache-ssl
          image: ubuntu:latest
          command:
            [
              "sh",
              "-c",
              "apt-get update && apt-get install -y apache2 && cp /setup-ssl.sh /setup-ssl.sh && chmod +x /setup-ssl.sh && /setup-ssl.sh",
            ]
          volumeMounts:
            - name: setup-script
              mountPath: /setup-ssl.sh
              subPath: setup-ssl.sh
            - name: ssl-certificates
              mountPath: /etc/ssl/certs/
      volumes:
        - name: custom-ca
          secret:
            secretName: custom-ca-secret
        - name: ca-certificates
          emptyDir: {}
        - name: ssl-certificates
          secret:
            secretName: wordpress-tls-secret
        - name: setup-script
          configMap:
            name: setup-script-configmap
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: setup-script-configmap
data:
  setup-ssl.sh: |
    #!/bin/bash
    a2enmod ssl
    a2enmod rewrite
    cat <<EOF > /etc/apache2/sites-available/wordpress-ssl.conf
    <VirtualHost *:443>
        ServerName your-domain.com
        DocumentRoot /var/www/html
        SSLEngine on
        SSLCertificateFile /etc/ssl/certs/tls.crt
        SSLCertificateKeyFile /etc/ssl/certs/tls.key
        <Directory /var/www/html>
            AllowOverride All
        </Directory>
    </VirtualHost>
    EOF
    a2ensite wordpress-ssl.conf
    service apache2 restart
---
apiVersion: v1
kind: Secret
metadata:
  name: custom-ca-secret
type: Opaque
data:
  ca.crt: <base64-encoded-ca-cert>
---
apiVersion: v1
kind: Secret
metadata:
  name: wordpress-tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-company-cert>
  tls.key: <base64-encoded-company-key>
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 443
      targetPort: 443
      protocol: TCP
      name: https
  selector:
    app: wordpress
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: wordpress-route
spec:
  host: your-domain.com # Replace with your domain
  to:
    kind: Service
    name: wordpress
  port:
    targetPort: https
  tls:
    termination: passthrough
```

### Key Points

1. **Custom Script**:

   - **Enable SSL**: Uses `a2enmod ssl` to enable the SSL module in Apache.
   - **Configuration**: Sets up a virtual host with `SSLCertificateFile` and `SSLCertificateKeyFile` directives pointing to your certificate and key.
   - **Script Execution**: Runs as an init container to configure Apache before starting the main WordPress container.

2. **Volume Mounts**:

   - **SSL Certificates**: Mounted into the container where Apache expects to find them.

3. **ConfigMap**:

   - **`setup-script-configmap`**: Contains the `setup-ssl.sh` script to be executed by the init container.

4. **Secrets**:

   - **`wordpress-tls-secret`**: Contains your company's SSL/TLS certificate and private key.
   - **`custom-ca-secret`**: Contains any additional CA certificates if needed.

5. **Service and Route**:
   - **Passthrough Termination**: The OpenShift Route is set to `passthrough`, ensuring the SSL/TLS connection is maintained from the client to the WordPress container.

### Base64 Encoding

To encode your certificate and key files:

```bash
base64 -w0 path/to/company-cert.crt
base64 -w0 path/to/company-key.key
```

### Summary

- **Certificate Management**: Store and use your company's SSL/TLS certificate and key within the WordPress container.
- **Route Configuration**: Use `passthrough` termination to ensure SSL/TLS is handled within the WordPress container.
- **Testing**: Verify that your WordPress instance is accessible via HTTPS and that SSL/TLS is correctly configured.

By following these steps, you will ensure that SSL/TLS is properly configured for your WordPress deployment using Apache within a private network.
