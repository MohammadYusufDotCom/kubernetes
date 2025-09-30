iled Documentation: Mounting a PFX SSL Certificate for Kubernetes Ingress TLS
Overview
This guide explains how to replace the default fake/self-signed certificate of a Kubernetes ingress controller with a real SSL certificate from a PFX file. It covers extracting the certificate and private key, creating the Kubernetes secret, updating ingress, and troubleshooting common issues.
Prerequisites
Access to Kubernetes cluster with the appropriate namespace (here automation-netcore-sql-poc)
PFX certificate file stored as a Kubernetes secret (pfx-secret)
PFX password
Kubectl CLI access with permissions to manage secrets and ingresses
OpenSSL installed on your local or cluster node machine
Step 1: Extract the PFX File from Kubernetes Secret
The PFX file is stored inside an opaque Kubernetes secret as a base64 encoded blob.
Command:


bashkubectl get secret pfx-secret \  -n automation-netcore-sql-poc \  -o jsonpath='{.data.businessbywire\.com\.pfx}' | base64 -d > certificate.pfx
Notes:
This exports the PFX to a file named certificate.pfx.
If you have a different secret name or key, adjust accordingly.
Step 2: Extract Certificate and Private Key from the PFX
The PFX is password protected (CRMnext@2026 in this case). You need the password to decrypt it.
Extract the certificate:


bashopenssl pkcs12 \  -in certificate.pfx \  -passin pass:CRMnext@2026 \  -clcerts \  -nokeys \  -out certificate.crt


Extract the private key:


bashopenssl pkcs12 \  -in certificate.pfx \  -passin pass:CRMnext@2026 \  -nocerts \  -nodes \  -out certificate.key


Clean up the private key:


bashopenssl rsa -in certificate.key -out certificate-clean.key


Verify:
Check the certificate PEM file:


bashhead -n5 certificate.crt
It should start with:


text-----BEGIN CERTIFICATE-----


Step 3: Create Kubernetes TLS Secret
Create a TLS type secret in your K8s namespace with certificate and private key:


bashkubectl create secret tls netcore-tls-secret \  --cert=certificate.crt \  --key=certificate-clean.key \  -n automation-netcore-sql-poc
Verify:


bashkubectl describe secret netcore-tls-secret -n automation-netcore-sql-poc


Step 4: Patch Your Ingress to Use the TLS Secret and Domain Host
Create a patch file named ingress-patch.yaml:


textspec:  tls:    - hosts:        - automation.businessbywire.com      secretName: netcore-tls-secret  rules:    - host: automation.businessbywire.com      http:        paths:          - path: /netapp            pathType: Prefix            backend:              service:                name: businessnext-app-svc                port:                  number: 443          # Add other backend paths here
Apply the patch:


bashkubectl patch ingress automation-netcore-ingress \  -n automation-netcore-sql-poc \  --type strategic \  --patch "$(cat ingress-patch.yaml)"


Step 5: Validate the Ingress Configuration and TLS Setup
Check the ingress resource to verify tls and host are correctly set:


bashkubectl get ingress automation-netcore-ingress -n automation-netcore-sql-poc -o yamlkubectl describe ingress automation-netcore-ingress -n automation-netcore-sql-poc
Test HTTPS from client/browser or CLI:


bashcurl -I https://automation.businessbywire.com/netapp
Expected: Response with 200 OK and the real certificate (no "fake" Kubernetes certificate warning).
Troubleshooting and Issues Encountered
Empty Certificate File: Initial extraction showed empty certificate.crt. This often happens if the PFX import password is missing or incorrect.
Invalid Password Error: Mac verify error: invalid password occurred until the correct password CRMnext@2026 was used.
Need for --legacy Flag: On some OpenSSL 3.x versions, the --legacy flag may be required for older PFX files (openssl pkcs12 --legacy ...).
Missing Host in Ingress Rules: The ingress resource initially lacked the host field in the rules, which caused Kubernetes to serve the default fake ingress certificate.
Correct Hostname Configuration: Ensuring host: automation.businessbywire.com in ingress rules and TLS hosts: is critical for proper TLS termination.
Kubernetes TLS Secret Requirements: The TLS secret must have PEM formatted tls.crt and tls.key. Using raw PFX or other formats will cause errors.
