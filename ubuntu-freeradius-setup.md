# FreeRADIUS Ubuntu Setup for WPA2 EAP Authentication with Android Devices

This guide provides comprehensive instructions for installing and configuring FreeRADIUS on Ubuntu Server to handle WPA2 EAP authentication for Android devices using Intune-distributed certificates.

## Prerequisites

- Ubuntu Server 20.04 LTS or 22.04 LTS
- Sudo privileges
- Access to your Intune-managed Android devices
- Your internal Root CA certificate (the one distributed by Intune)
- Basic Linux command line knowledge

## Step 1: System Preparation

### 1.1 Update System Packages

```bash
sudo apt update
sudo apt upgrade -y
```

### 1.2 Install Required Packages

```bash
sudo apt install -y freeradius freeradius-mysql freeradius-utils openssl
```

### 1.3 Create Backup Directory

```bash
sudo mkdir -p /etc/raddb/backup
sudo cp -r /etc/raddb/* /etc/raddb/backup/
```

## Step 2: Configure FreeRADIUS

### 2.1 Configure Main Settings

Edit the main configuration file:

```bash
sudo nano /etc/freeradius/3.0/radiusd.conf
```

Update the following sections:

```conf
prefix = /usr
exec_prefix = /usr
sysconfdir = /etc
localstatedir = /var
sbindir = ${exec_prefix}/sbin
logdir = /var/log/freeradius
raddbdir = /etc/freeradius/3.0
radacctdir = ${logdir}/radacct

name = freeradius
confdir = ${raddbdir}
modconfdir = ${confdir}/mods-config
certdir = ${confdir}/certs
cadir   = ${confdir}/certs
run_dir = ${localstatedir}/run/${name}

db_dir = ${raddbdir}

libdir = /usr/lib/freeradius

pidfile = ${run_dir}/${name}.pid

correct_escapes = true

max_request_time = 30

hostname_lookups = no

allow_core_dumps = no

regular_expressions = yes
extended_expressions = yes

log {
        destination = files
        file = ${logdir}/radius.log
        syslog_facility = daemon
        stripped_names = no
        auth = yes
        auth_badpass = no
        auth_goodpass = no
}

checkrad = ${sbindir}/checkrad

security {
        max_attributes = 200
        reject_delay = 1
        status_server = yes
        allow_core_dumps = no
}

proxy_requests  = yes
$INCLUDE proxy.conf

$INCLUDE clients.conf

thread pool {
        start_servers = 5
        max_servers = 32
        min_spare_servers = 3
        max_spare_servers = 10
        max_requests_per_server = 0
}

modules {
        $INCLUDE ${confdir}/mods-enabled/
}

instantiate {
        exec
        expr
        expiration
        logintime
        pap
}

$INCLUDE policy.conf

$INCLUDE sites-enabled/
```

### 2.2 Configure Clients

Edit the clients configuration:

```bash
sudo nano /etc/freeradius/3.0/clients.conf
```

Add your wireless controller configuration:

```conf
client localhost {
        ipaddr = 127.0.0.1
        secret = testing123
        require_message_authenticator = no
        nas_type = other
}

# Add your wireless controller/AP here
client your_wireless_controller {
        ipaddr = YOUR_WIRELESS_CONTROLLER_IP
        secret = your_radius_secret_here
        require_message_authenticator = no
        nas_type = other
}
```

### 2.3 Configure EAP-TLS

Edit the EAP configuration:

```bash
sudo nano /etc/freeradius/3.0/mods-available/eap
```

Update the configuration:

```conf
eap {
        default_eap_type = tls
        timer_expire     = 60
        ignore_unknown_eap_types = no
        cisco_accounting_username_bug = no
        max_sessions = 4096

        tls {
                private_key_password = whatever
                private_key_file = ${certdir}/server.key
                certificate_file = ${certdir}/server.crt
                ca_file = ${certdir}/ca.crt
                dh_file = ${certdir}/dh
                random_file = /dev/urandom
                fragment_size = 1024
                include_length = yes
                check_crl = no
                check_cert_cn = %{User-Name}
                cipher_list = "DEFAULT"
                ecdh_curve = "prime256v1"
                cache {
                        enable = yes
                        lifetime = 24 # hours
                        max_entries = 255
                }
                verify {
                        tmpdir = /tmp/radiusd
                        client = "/usr/bin/openssl verify -CAfile ${..ca_file} %{TLS-Client-Cert-Filename}"
                }
        }
}
```

### 2.4 Configure Default Site

Edit the default site configuration:

```bash
sudo nano /etc/freeradius/3.0/sites-available/default
```

Update the configuration:

```conf
server default {
        listen {
                type = auth
                ipaddr = *
                port = 0
                limit {
                        max_connections = 16
                        lifetime = 0
                        idle_timeout = 30
                }
        }

        authorize {
                eap
        }

        authenticate {
                Auth-Type EAP {
                        eap
                }
        }

        preacct {
                preprocess
                acct_unique
                suffix
        }

        accounting {
                detail
                unix
                exec
                attr_filter.accounting_response
        }

        session {
                radutmp
        }

        post-auth {
                update {
                        &reply: += &session-state:
                }
                exec
                remove_reply_message_if_eap
                Post-Auth-Type REJECT {
                        attr_filter.access_reject
                        eap
                }
        }
}
```

## Step 3: Prepare Certificates

### 3.1 Copy Your Root CA Certificate

```bash
# Copy your Intune Root CA certificate
sudo cp /path/to/your/intune-root-ca.crt /etc/freeradius/3.0/certs/ca.crt
```

### 3.2 Generate Server Certificate

You have two options for generating the server certificate:

#### Option A: Using Microsoft CA (Recommended for Enterprise)

This approach uses your existing Microsoft CA infrastructure for consistency.

1. **Generate Certificate Request on Windows Server with Microsoft CA:**

```powershell
# On your Windows Server with Microsoft CA installed
certreq -new -f -q -attrib "SAN=DNS=freeradius-server.yourdomain.com&DNS=freeradius-server&IPAddress=YOUR_SERVER_IP" C:\temp\freeradius.inf C:\temp\freeradius.req
```

Create the `freeradius.inf` file:

```ini
[NewRequest]
Subject = "CN=freeradius-server, O=YourOrganization, C=US"
KeySpec = 1
KeyLength = 2048
Exportable = TRUE
MachineKeySet = TRUE
SMIME = FALSE
PrivateKeyArchive = FALSE
UserProtected = FALSE
UseExistingKeySet = FALSE
ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
ProviderType = 12
RequestType = PKCS10
KeyUsage = 0xa0
[EnhancedKeyUsageExtension]
OID=1.3.6.1.5.5.7.3.1
[Extensions]
2.5.29.17 = "{text}"
_continue_ = "DNS=freeradius-server.yourdomain.com&"
_continue_ = "DNS=freeradius-server&"
_continue_ = "IPAddress=YOUR_SERVER_IP"
```

2. **Submit Request to Microsoft CA:**

```powershell
# Submit the request to your Microsoft CA
certreq -submit -attrib "CertificateTemplate=WebServer" C:\temp\freeradius.req C:\temp\freeradius.cer
```

3. **Export Private Key and Certificate:**

```powershell
# Install the certificate
certreq -accept C:\temp\freeradius.cer

# Export the certificate with private key
certutil -exportpfx -p "YourPassword" My "freeradius-server" C:\temp\freeradius.pfx
```

4. **Transfer and Convert on Ubuntu Server:**

```bash
# Copy the PFX file to your Ubuntu server
scp user@windows-server:/temp/freeradius.pfx /tmp/

# Convert PFX to PEM files
sudo openssl pkcs12 -in /tmp/freeradius.pfx -out /etc/freeradius/3.0/certs/server.key -nocerts -nodes -passin pass:YourPassword
sudo openssl pkcs12 -in /tmp/freeradius.pfx -out /etc/freeradius/3.0/certs/server.crt -clcerts -nokeys -passin pass:YourPassword

# Clean up the private key file (remove the header/footer)
sudo sed -i '/^-----BEGIN PRIVATE KEY-----$/,$!d;/^-----END PRIVATE KEY-----$/q' /etc/freeradius/3.0/certs/server.key

# Clean up temporary files
rm /tmp/freeradius.pfx
```

#### Option B: Local Certificate Generation

Create a script to generate the server certificate locally:

```bash
sudo nano /root/generate-certs.sh
```

Add the following content:

```bash
#!/bin/bash

cd /etc/freeradius/3.0/certs

# Generate server private key
openssl genrsa -out server.key 2048

# Generate server certificate signing request
openssl req -new -key server.key -out server.csr -subj "/CN=freeradius-server/O=YourOrganization/C=US"

# Sign the certificate with your Root CA
openssl x509 -req -in server.csr -CA ca.crt -CAkey /path/to/your/ca.key -CAcreateserial -out server.crt -days 365

# Generate DH parameters
openssl dhparam -out dh 2048

# Set proper permissions
chmod 600 server.key
chmod 644 server.crt ca.crt dh

# Clean up
rm server.csr

echo "Certificate generation completed!"
```

Make the script executable and run it:

```bash
sudo chmod +x /root/generate-certs.sh
sudo /root/generate-certs.sh
```

### 3.3 Generate DH Parameters (Required for both options)

```bash
# Generate DH parameters for TLS
sudo openssl dhparam -out /etc/freeradius/3.0/certs/dh 2048
sudo chmod 644 /etc/freeradius/3.0/certs/dh
```

### 3.4 Verify Certificate Chain

```bash
# Verify the certificate chain
sudo openssl verify -CAfile /etc/freeradius/3.0/certs/ca.crt /etc/freeradius/3.0/certs/server.crt

# Check certificate details
sudo openssl x509 -in /etc/freeradius/3.0/certs/server.crt -text -noout
```

## Step 4: Configure Firewall

### 4.1 Allow RADIUS Ports

```bash
sudo ufw allow 1812/udp
sudo ufw allow 1813/udp
sudo ufw allow 18120/tcp
sudo ufw enable
```

### 4.2 Verify Firewall Status

```bash
sudo ufw status
```

## Step 5: Start and Enable FreeRADIUS

### 5.1 Test Configuration

```bash
sudo freeradius -X
```

If the configuration is correct, you should see "Ready to process requests" at the end.

### 5.2 Start FreeRADIUS Service

```bash
sudo systemctl start freeradius
sudo systemctl enable freeradius
```

### 5.3 Check Service Status

```bash
sudo systemctl status freeradius
```

## Step 6: Test the Configuration

### 6.1 Test from Localhost

```bash
radtest -t eap-tls -x testing123 127.0.0.1 0
```

### 6.2 Test with Android Device

1. On your Android device, go to Settings > Wi-Fi
2. Add a new network with the following settings:
   - SSID: Your network name
   - Security: WPA2 Enterprise
   - EAP method: TLS
   - CA certificate: Select your Intune-distributed Root CA
   - Client certificate: Select the SCEP certificate distributed by Intune
   - Identity: Leave blank or use a device identifier

## Step 7: Configure Your Wireless Controller

Configure your wireless controller/access point to use the FreeRADIUS server:

- RADIUS Server IP: Your Ubuntu server IP
- RADIUS Port: 1812
- RADIUS Secret: The secret you configured in `clients.conf`
- Authentication Method: EAP-TLS

## Step 8: Monitor and Troubleshoot

### 8.1 View Logs

```bash
# View FreeRADIUS logs
sudo tail -f /var/log/freeradius/radius.log

# View system logs
sudo journalctl -u freeradius -f
```

### 8.2 Common Issues and Solutions

1. **Certificate Issues**: Ensure your Root CA certificate is properly formatted and trusted
2. **Client Configuration**: Verify the wireless controller IP is correctly configured in `clients.conf`
3. **Network Connectivity**: Ensure ports 1812/udp and 1813/udp are accessible from your wireless controller
4. **Permission Issues**: Ensure certificate files have proper permissions

## Step 9: Security Considerations

### 9.1 Change Default Secrets

```bash
sudo nano /etc/freeradius/3.0/clients.conf
```

Replace `testing123` with a strong secret.

### 9.2 Secure Certificate Files

```bash
sudo chown -R freerad:freerad /etc/freeradius/3.0/certs/
sudo chmod 600 /etc/freeradius/3.0/certs/server.key
sudo chmod 644 /etc/freeradius/3.0/certs/server.crt
sudo chmod 644 /etc/freeradius/3.0/certs/ca.crt
```

### 9.3 Regular Updates

```bash
sudo apt update
sudo apt upgrade freeradius
```

## Step 10: Backup and Maintenance

### 10.1 Create Backup Script

```bash
sudo nano /root/backup-freeradius.sh
```

Add the following content:

```bash
#!/bin/bash

BACKUP_DIR="/backup/freeradius"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup configuration
tar -czf $BACKUP_DIR/freeradius-config-$DATE.tar.gz /etc/freeradius/3.0/

# Backup certificates
tar -czf $BACKUP_DIR/freeradius-certs-$DATE.tar.gz /etc/freeradius/3.0/certs/

# Backup logs
tar -czf $BACKUP_DIR/freeradius-logs-$DATE.tar.gz /var/log/freeradius/

echo "Backup completed: $BACKUP_DIR"
```

Make it executable:

```bash
sudo chmod +x /root/backup-freeradius.sh
```

### 10.2 Set Up Automated Backups

```bash
sudo crontab -e
```

Add the following line for daily backups:

```
0 2 * * * /root/backup-freeradius.sh
```

## Step 11: Performance Tuning

### 11.1 Optimize for High Load

Edit the main configuration:

```bash
sudo nano /etc/freeradius/3.0/radiusd.conf
```

Update the thread pool settings:

```conf
thread pool {
        start_servers = 10
        max_servers = 64
        min_spare_servers = 5
        max_spare_servers = 20
        max_requests_per_server = 0
}
```

### 11.2 Monitor Performance

```bash
# Check FreeRADIUS status
sudo radmin -e "show status"

# Monitor system resources
htop
```

## Troubleshooting Commands

```bash
# Check FreeRADIUS status
sudo systemctl status freeradius

# View real-time logs
sudo tail -f /var/log/freeradius/radius.log

# Test configuration
sudo freeradius -X

# Check certificate validity
sudo openssl x509 -in /etc/freeradius/3.0/certs/server.crt -text -noout

# Check network connectivity
sudo netstat -tulpn | grep 1812

# Check firewall rules
sudo ufw status

# Restart FreeRADIUS
sudo systemctl restart freeradius
```

## Additional Resources

- FreeRADIUS Documentation: https://freeradius.org/documentation/
- Ubuntu Server Guide: https://ubuntu.com/server/docs
- OpenSSL Documentation: https://www.openssl.org/docs/

This setup provides a robust FreeRADIUS server that validates certificates distributed by Intune without requiring AD attributes, making it perfect for your Android kiosk mode deployment.

## Key Benefits of Using Microsoft CA

1. **Centralized Management**: All certificates are managed through your existing CA infrastructure
2. **Consistency**: Same certificate templates and policies across your organization
3. **Automated Renewal**: Can leverage existing certificate auto-renewal processes
4. **Audit Trail**: Certificate requests and issuance are logged in your CA
5. **Integration**: Works seamlessly with your existing PKI infrastructure

## Important Considerations

1. **Certificate Template**: Use a template that supports server authentication (like "WebServer")
2. **Subject Alternative Names (SAN)**: Include both DNS names and IP addresses for flexibility
3. **Key Usage**: Ensure the certificate has the correct key usage extensions
4. **Validity Period**: Match your organization's certificate lifecycle policies
5. **Private Key Protection**: Use strong passwords when exporting the PFX file

The Microsoft CA approach is definitely the recommended method for enterprise environments, as it maintains consistency with your existing certificate infrastructure while providing the same functionality for FreeRADIUS authentication.
