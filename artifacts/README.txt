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
# only deploying Busybox.
#
# Each of the first four patterns must match EXACTLY ONE file. The workload playbooks
# fail with the list of matching files if there are two or more (for example an older
# client or SE header left over from a previous bundle). Remove the extra files and re-run.
#
# The sealed secret client and the SE header (se-hdr.bin) also ship inside the IBM fixpack
# bundle (*.tar.gz). Today you place them here yourself. The automated image push
# (playbooks/ccco_push_image.yaml, currently disabled in ccco_install_cluster.yaml) extracts
# them from the bundle into this directory when it runs; it does not remove older copies.
# Keep at most one *.tar.gz here (the bundle), and delete any older client or .bin first.
#
# No TLS certificate handy? Generate a self-signed pair for testing:
#
#   openssl req -x509 -newkey rsa:4096 -nodes -days 365 \
#     -subj "/CN=<your-hostname>" \
#     -keyout artifacts/common_app_key.pem \
#     -out artifacts/common_app_cert.pem
#
# This directory is git-ignored. Do NOT commit secrets or certificates.
