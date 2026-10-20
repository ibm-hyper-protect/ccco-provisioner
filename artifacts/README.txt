# Place the following files here before running the playbook:
#
#   HKD*.crt                                   - Host Key Document certificate
#   <contract_encryption_key>.crt              - IBM CCCO contract encryption certificate (any non-HKD *.crt)
#   <se_header>.bin                            - SE header binary
#   ibm-ccco-sealed-secret-creator-client-*    - sealed secret client binary (s390x)
#   common_app_cert.pem                        - TLS certificate for PayNow application (CERT env var)
#   common_app_key.pem                         - TLS private key for PayNow application (KEY env var)
#
# All files are required, including the two common_app_*.pem files, even when
# only deploying Busybox. The first match per pattern is used.
#
# This directory is git-ignored. Do NOT commit secrets or certificates.
