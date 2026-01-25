# Ansible Role: ACME Certificate Issue

An Ansible role for automated ACME certificate issuance and renewal using the http-01 challenge method. Works with any ACME-compatible CA including Let's Encrypt, step-ca, and others.

## Features

- **Automated Certificate Management**: Issue and renew SSL/TLS certificates automatically
- **http-01 Challenge**: Uses webroot verification for domain ownership
- **Smart Renewal**: Only renews certificates when approaching expiration
- **ECDSA Support**: Generates modern ECDSA P-256 keys by default
- **Flexible Configuration**: Works with any ACME-compatible CA
- **Idempotent**: Safe to run repeatedly, only acts when needed
- **Service Integration**: Automatically reloads web server on certificate changes

## Requirements

- Ansible 2.14+
- Collections:
  - `community.crypto` >= 2.0.0
- Webroot directory accessible for ACME challenge files
- Valid ACME account key
- DNS records pointing to the target host

## Role Variables

### Required Variables

These variables **must** be provided (no defaults):

```yaml
# Certificate Details
cert_common_name: "example.com"        # Primary domain name for certificate
cert_sans: []                           # List of all SANs including common name
cert_service_name: "myservice"         # Unique identifier for this certificate set
```

### Optional Variables (with defaults)

```yaml
# Step-CA Server Configuration
step_ca_host: "ca.example.com"         # CA server hostname
step_ca_port: 8443                     # CA server port

# ACME Server Configuration
acme_directory_url: "https://ca.example.com/acme/directory"
acme_account_key_path: "/etc/ssl/private/acme_account.key"

# File Paths and Permissions
cert_webroot: "/var/www/html"          # Webroot for ACME challenge
cert_install_dir: "/etc/ssl"           # Directory for certificate files
cert_owner: "root"                     # Owner of certificate files
cert_group: "root"                     # Group of certificate files

# Service Configuration
web_server_service_name: "nginx"       # Service to reload on cert change

# Certificate Renewal Settings
cert_renewal_threshold_days: 30        # Renew when < N days remaining
cert_force_renewal: false              # Force renewal (for testing)
```

### Generated Files

The role creates the following files in `cert_install_dir`:

- `{{ cert_service_name }}.key` - Private key (ECDSA P-256)
- `{{ cert_service_name }}.csr` - Certificate signing request
- `{{ cert_service_name }}.crt` - Server certificate
- `{{ cert_service_name }}-chain.crt` - CA chain
- `{{ cert_service_name }}-fullchain.crt` - Full chain (cert + chain)

### Output Variables

The role sets the following fact on certificate renewal:

- `acme_cert_changed: true` - Set when certificate was renewed

## Dependencies

Install required collections:

```bash
ansible-galaxy collection install community.crypto
```

## Example Playbook

### Basic Usage with Let's Encrypt

```yaml
- hosts: webservers
  become: true
  roles:
    - role: ansible_role_acme_issue
      cert_common_name: "www.example.com"
      cert_sans:
        - "www.example.com"
        - "example.com"
      cert_service_name: "myapp"
      acme_directory_url: "https://acme-v02.api.letsencrypt.org/directory"
      cert_webroot: "/var/www/myapp"
      cert_owner: "www-data"
      cert_group: "www-data"
```

### Usage with step-ca (Internal CA)

```yaml
- hosts: internal_servers
  become: true
  vars:
    step_ca_host: "ca.internal.example.com"
    step_ca_port: 8443
    acme_directory_url: "https://{{ step_ca_host }}:{{ step_ca_port }}/acme/acme/directory"
  
  roles:
    - role: ansible_role_acme_issue
      cert_common_name: "app.internal.example.com"
      cert_sans:
        - "app.internal.example.com"
        - "192.168.1.100"
      cert_service_name: "internal-app"
      cert_webroot: "/opt/myapp/public"
      web_server_service_name: "apache2"
```

### Automated Renewal with Post-Tasks

```yaml
- hosts: webservers
  become: true
  
  roles:
    - role: ansible_role_acme_issue
      cert_service_name: "nextcloud"
      cert_common_name: "cloud.example.com"
      cert_sans:
        - "cloud.example.com"
      cert_renewal_threshold_days: 30
  
  post_tasks:
    - name: Reload nginx if certificate changed
      ansible.builtin.service:
        name: nginx
        state: reloaded
      when: acme_cert_changed | default(false)
    
    - name: Send notification on renewal
      ansible.builtin.debug:
        msg: "Certificate renewed at {{ ansible_date_time.iso8601 }}"
      when: acme_cert_changed | default(false)
```

### Using in a Cron Job (Renewal Playbook)

```yaml
# renew_certs.yml
- hosts: webservers
  become: true
  
  roles:
    - role: ansible_role_acme_issue
      cert_service_name: "myapp"
      cert_common_name: "app.example.com"
      cert_sans:
        - "app.example.com"
      cert_renewal_threshold_days: 30
```

Schedule with cron:
```bash
# Run weekly: Sundays at 3:00 AM
0 3 * * 0 /usr/bin/ansible-playbook /path/to/renew_certs.yml
```

## How It Works

1. **CA Trust Setup**: Fetches and installs the CA root certificate
2. **Challenge Preparation**: Creates webroot challenge directory
3. **Certificate Check**: Examines existing certificate expiration
4. **Smart Renewal Decision**: Only proceeds if:
   - No certificate exists, OR
   - Certificate expires within threshold, OR
   - Force renewal is enabled
5. **Key Generation**: Creates ECDSA P-256 private key (if needed)
6. **CSR Creation**: Generates certificate signing request
7. **ACME Registration**: Registers/verifies ACME account
8. **Order Creation**: Creates certificate order with CA
9. **Challenge Fulfillment**: Publishes http-01 challenge tokens to webroot
10. **Validation**: Validates challenges with CA
11. **Finalization**: Retrieves signed certificate and chain
12. **Installation**: Installs certificate files with proper permissions
13. **Fact Setting**: Sets `acme_cert_changed` for downstream tasks

## ACME Account Setup

Before using this role, create an ACME account key:

```bash
# Generate ACME account key
openssl genrsa -out /etc/ssl/private/acme_account.key 4096
chmod 600 /etc/ssl/private/acme_account.key
```

Or use Ansible:

```yaml
- name: Generate ACME account key
  community.crypto.openssl_privatekey:
    path: /etc/ssl/private/acme_account.key
    type: RSA
    size: 4096
    mode: "0600"
```

## Webroot Configuration

Ensure your web server serves the ACME challenge directory:

### Nginx

```nginx
server {
    listen 80;
    server_name example.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/html;
        default_type text/plain;
    }
}
```

### Apache

```apache
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/html
    
    <Directory /var/www/html/.well-known/acme-challenge>
        Require all granted
    </Directory>
</VirtualHost>
```

## Troubleshooting

### Certificate Not Renewing

**Check expiration:**
```bash
openssl x509 -in /etc/ssl/myservice.crt -noout -enddate
```

**Force renewal for testing:**
```yaml
cert_force_renewal: true
```

### Challenge Validation Fails

**Verify webroot accessibility:**
```bash
curl http://example.com/.well-known/acme-challenge/test
```

**Check permissions:**
```bash
ls -la /var/www/html/.well-known/acme-challenge/
```

### CA Trust Issues

**Manually verify CA connection:**
```bash
curl https://ca.example.com:8443/roots.pem
```

**Check installed CA certificates:**
```bash
ls -la /usr/local/share/ca-certificates/
```

### DNS Not Resolving

Ensure DNS records point to the correct server:
```bash
dig +short example.com
nslookup example.com
```

### ACME Account Issues

Verify account key exists and has correct permissions:
```bash
ls -la /etc/ssl/private/acme_account.key
```

## Security Considerations

- **Private Keys**: Always protect private keys (0600/0640 permissions)
- **Account Keys**: Store ACME account keys securely
- **Webroot Security**: Ensure challenge directory is publicly accessible but secure
- **CA Trust**: Validate CA certificates before trusting
- **HTTPS Only**: Redirect HTTP to HTTPS after certificate installation

## License

MIT

## Author Information

Created for use with step-ca and Let's Encrypt certificate automation.

## Contributing

Issues and pull requests welcome! Please follow Ansible best practices and test thoroughly.
