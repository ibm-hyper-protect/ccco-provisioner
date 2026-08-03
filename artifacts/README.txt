# Place the following files here before running the playbook:
#
#   HKD*.crt                              - Host Key Document certificate
#   <contract_encryption_key>.crt         - IBM CCCO contract encryption certificate
#   <se_header>.bin                       - SE header binary
#   <sealed_secret_client>                - ibm-ccco-sealed-secret-creator-client
#   PayNow_app_cert.pem                   - TLS certificate for PayNow application (CERT env var)
#   PayNow_app_key.pem                    - TLS private key for PayNow application (KEY env var)
#
# This directory is git-ignored. Do NOT commit secrets or certificates.
