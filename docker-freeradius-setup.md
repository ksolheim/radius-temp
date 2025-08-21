# FreeRADIUS Docker Setup for WPA2 EAP Authentication with Android Devices

This guide provides step-by-step instructions for setting up FreeRADIUS in Docker to handle WPA2 EAP authentication for Android devices using Intune-distributed certificates.

## Prerequisites

- Docker and Docker Compose installed on your system
- Access to your Intune-managed Android devices
- Your internal Root CA certificate (the one distributed by Intune)
- Basic understanding of Docker concepts

## Step 1: Create Project Directory

First, create a directory for your FreeRADIUS setup:

```bash
mkdir freeradius-docker
cd freeradius-docker
```

## Step 2: Create Docker Compose Configuration

Create a `docker-compose.yml` file with the following content:

```yaml
version: '3.8'

services:
  freeradius:
    image: freeradius/freeradius-server:latest
    container_name: freeradius-server
    ports:
      - "1812:1812/udp"  # RADIUS authentication
      - "1813:1813/udp"  # RADIUS accounting
      - "18120:18120"    # FreeRADIUS status
    volumes:
      - ./config:/etc/raddb
      - ./certs:/etc/raddb/certs
      - ./logs:/var/log/freeradius
    environment:
      - RADIUS_SECRET=your_radius_secret_here
    restart: unless-stopped
    command: ["freeradius", "-X", "-f"]
```

## Step 3: Create Configuration Directory Structure

Create the necessary directories and configuration files:

```bash
mkdir -p config certs logs
```

## Step 4: Configure FreeRADIUS

### 4.1 Create Main Configuration File

Create `config/radiusd.conf`:

```conf
prefix = /usr
exec_prefix = /usr
sysconfdir = /etc
localstatedir = /var
sbindir = ${exec_prefix}/sbin
logdir = /var/log/freeradius
raddbdir = /etc/raddb
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

### 4.2 Configure Clients

Create `config/clients.conf`:

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

### 4.3 Configure EAP-TLS

Create `config/mods-available/eap`:

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

### 4.4 Configure Authentication

Create `config/sites-available/default`:

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

## Step 5: Prepare Certificates

### 5.1 Copy Your Root CA Certificate

Place your Intune-distributed Root CA certificate in the `certs` directory:

```bash
# Copy your Root CA certificate to certs/ca.crt
cp /path/to/your/intune-root-ca.crt certs/ca.crt
```

### 5.2 Generate Server Certificate

Create a script to generate the server certificate (`generate-certs.sh`):

```bash
#!/bin/bash

# Generate server private key
openssl genrsa -out certs/server.key 2048

# Generate server certificate signing request
openssl req -new -key certs/server.key -out certs/server.csr -subj "/CN=freeradius-server/O=YourOrganization/C=US"

# Sign the certificate with your Root CA
openssl x509 -req -in certs/server.csr -CA certs/ca.crt -CAkey /path/to/your/ca.key -CAcreateserial -out certs/server.crt -days 365

# Generate DH parameters
openssl dhparam -out certs/dh 2048

# Set proper permissions
chmod 600 certs/server.key
chmod 644 certs/server.crt certs/ca.crt certs/dh
```

Make the script executable and run it:

```bash
chmod +x generate-certs.sh
./generate-certs.sh
```

## Step 6: Start FreeRADIUS

Start the FreeRADIUS container:

```bash
docker-compose up -d
```

Check if the container is running:

```bash
docker-compose ps
docker-compose logs freeradius
```

## Step 7: Test the Configuration

### 7.1 Test from Inside the Container

```bash
docker exec -it freeradius-server bash
radtest -t eap-tls -x testing123 127.0.0.1 0
```

### 7.2 Test with Android Device

1. On your Android device, go to Settings > Wi-Fi
2. Add a new network with the following settings:
   - SSID: Your network name
   - Security: WPA2 Enterprise
   - EAP method: TLS
   - CA certificate: Select your Intune-distributed Root CA
   - Client certificate: Select the SCEP certificate distributed by Intune
   - Identity: Leave blank or use a device identifier

## Step 8: Configure Your Wireless Controller

Configure your wireless controller/access point to use the FreeRADIUS server:

- RADIUS Server IP: Your Docker host IP
- RADIUS Port: 1812
- RADIUS Secret: The secret you configured in `clients.conf`
- Authentication Method: EAP-TLS

## Step 9: Monitor and Troubleshoot

### 9.1 View Logs

```bash
# View FreeRADIUS logs
docker-compose logs -f freeradius

# View detailed logs from inside the container
docker exec -it freeradius-server tail -f /var/log/freeradius/radius.log
```

### 9.2 Common Issues and Solutions

1. **Certificate Issues**: Ensure your Root CA certificate is properly formatted and trusted
2. **Client Configuration**: Verify the wireless controller IP is correctly configured in `clients.conf`
3. **Network Connectivity**: Ensure ports 1812/udp and 1813/udp are accessible from your wireless controller

## Step 10: Security Considerations

1. Change default secrets in `clients.conf`
2. Use strong passwords for private keys
3. Regularly update certificates
4. Monitor logs for suspicious activity
5. Consider implementing certificate revocation checking

## Step 11: Backup and Maintenance

### 11.1 Backup Configuration

```bash
tar -czf freeradius-backup-$(date +%Y%m%d).tar.gz config/ certs/ docker-compose.yml
```

### 11.2 Update FreeRADIUS

```bash
docker-compose pull
docker-compose down
docker-compose up -d
```

## Troubleshooting Commands

```bash
# Check container status
docker-compose ps

# View real-time logs
docker-compose logs -f

# Access container shell
docker exec -it freeradius-server bash

# Test RADIUS configuration
docker exec -it freeradius-server freeradius -X

# Check certificate validity
docker exec -it freeradius-server openssl x509 -in /etc/raddb/certs/server.crt -text -noout
```

This setup provides a robust FreeRADIUS server that validates certificates distributed by Intune without requiring AD attributes, making it perfect for your Android kiosk mode deployment.
