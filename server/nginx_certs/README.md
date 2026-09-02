# Self-Signed Certificate Generation

This directory contains configuration and instructions for generating self-signed SSL/TLS certificates for local development and testing, specifically configured for use with Grafana and Nginx.

The certificate generated here is for the Grafana Web Nginx Proxy so TLS is used when logging into the wen interface.

NOTE: the cert and key so need to be name server.crt and server.key. they also need to be in this folder for the docker compose to pull them into the containers.

## Overview

The `openssl.conf` file contains the OpenSSL configuration needed to generate a self-signed certificate with Subject Alternative Names (SANs) and proper extensions for server authentication.

## Certificate Details

The current `openssl.conf` is configured with the following details:

| Property | Value |
|----------|-------|
| **Key Size** | 2048 bits |
| **Common Name (CN)** | grafana.domain.com |
| **Organization** | My Organization |
| **Organization Unit** | IT Department |
| **Country** | US |
| **State** | Arizona |
| **City** | Tucson |
| **Subject Alternative Names** | grafana.domain.com, grafana, localhost, 127.0.0.1 |
| **Key Usage** | Digital Signature, Key Encipherment |
| **Extended Key Usage** | Server Auth, Client Auth |

## Prerequisites

Ensure OpenSSL is installed on your Linux system:

```bash
# For Debian/Ubuntu
sudo apt-get install openssl

# For RHEL/CentOS
sudo yum install openssl

# For Alpine
apk add openssl

# Verify installation
openssl version
```

## Customizing the Certificate (Optional)

Before generating the certificate, you may want to customize the certificate details:

1. Open `openssl.conf` in a text editor
2. Update the `[ req_distinguished_name ]` section with your information:
   - `C` - Country code (e.g., US)
   - `ST` - State or Province
   - `L` - City/Locality
   - `O` - Organization name
   - `OU` - Organizational Unit
   - `CN` - Common Name (primary domain)
   - `emailAddress` - Contact email

3. Update the `[ alt_names ]` section with your domain names and IP addresses:
   ```
   DNS.1 = localhost
   DNS.2 = yourdomain.com
   DNS.3 = www.yourdomain.com
   IP.1 = 192.168.1.100
   ```

## Generating the Self-Signed Certificate

Run the following command in the `nginx_certs` directory to generate a self-signed certificate and private key:

```bash
openssl req -x509 -new -nodes -keyout server.key -out server.crt -days 365 -config openssl.conf
```

### Command Breakdown

| Parameter | Purpose |
|-----------|---------|
| `req` | Certificate request processing |
| `-x509` | Output a self-signed certificate instead of a signing request |
| `-new` | Generate a new certificate request |
| `-nodes` | Don't encrypt the private key |
| `-keyout server.key` | Output file for the private key |
| `-out server.crt` | Output file for the certificate |
| `-days 365` | Certificate validity period (365 days = 1 year) |
| `-config openssl.conf` | Use this configuration file |

### Output Files

After running the command, you'll have two new files:

- **`server.crt`** - The self-signed certificate file (public)
- **`server.key`** - The private key file (keep secure)

## Verifying the Certificate

To view the certificate details and verify it was created correctly:

```bash
openssl x509 -in server.crt -text -noout
```

To check the certificate's Subject Alternative Names:

```bash
openssl x509 -in server.crt -noout -text | grep -A 1 "Subject Alternative Name"
```

## Additional Resources

- [OpenSSL Documentation](https://www.openssl.org/docs/)
- [Let's Encrypt (for production)](https://letsencrypt.org/)
- [Nginx SSL Configuration](https://nginx.org/en/docs/http/ngx_http_ssl_module.html)
- [Self-Signed Certificates - Mozilla](https://developer.mozilla.org/en-US/docs/Glossary/Self-signed_certificate)

## License

These configuration files and instructions are provided as-is for development and testing purposes.
