
# Minimal install, configure and unseal a Vault Server running on Ubuntu (generic x64 bit)


The scope of this repository is to provide the steps for install, configure and unseal a Vault server
generate-root-tokens-using-unseal-keys

-----
## Which are the main tools used to accomplish this task?

----
## Vault
-	Website: https://www.vaultproject.io
-	Announcement list: [Google Groups](https://groups.google.com/group/hashicorp-announce)
-	Discussion forum: [Discuss](https://discuss.hashicorp.com/c/vault)
- Documentation: [https://www.vaultproject.io/docs/](https://www.vaultproject.io/docs/)
- Tutorials: [HashiCorp's Learn Platform](https://learn.hashicorp.com/vault)
- Certification Exam: [Vault Associate](https://www.hashicorp.com/certification/#hashicorp-certified-vault-associate)

<img width="300" alt="Vault Logo" src="https://github.com/hashicorp/vault/blob/f22d202cde2018f9455dec755118a9b84586e082/Vault_PrimaryLogo_Black.png">

----
Vault is a tool for securely accessing secrets. A secret is anything that you want to tightly control access to, such as API keys, passwords, certificates, and more. Vault provides a unified interface to any secret, while providing tight access control and recording a detailed audit log.

----
**Please note**: We take Vault's security and our users' trust very seriously. If you believe you have found a security issue in Vault, _please responsibly disclose_ by contacting us at [security@hashicorp.com](mailto:security@hashicorp.com).

----


1. Creating the directory structure for a minimal installation of a Vault server
```shell
for i in $(echo '/root/kit /opt/vault /opt/vault/vault-data /opt/vault/vault-log' ) ; do
    echo "mkdir -p $i" ; mkdir -p ${i}
done
```

2. [Download the latest vault server](https://www.vaultproject.io/downloads)
```shell
apt update -y && apt install wget sudo netcat telnet zip
cd /root/kit 
wget https://releases.hashicorp.com/vault/1.11.0/vault_1.11.0_linux_amd64.zip
```


3. Unpack the latest version 
```
unzip vault_*_linux_amd64.zip && cp vault /usr/bin/vault
```

4. Create the Vault configuration file /opt/vault/vault-server.hcl in HCL format
```
tee /opt/vault/vault-server.hcl  << EOFC
api_addr                = "http://127.0.0.1:8200"
cluster_addr            = "http://127.0.0.1:8201"
cluster_name            = "learn-vault-cluster"
default_lease_ttl       = "10h"
disable_mlock           = true
max_lease_ttl           = "10h"

listener "tcp" {
  address       = "127.0.0.1:8200"
  tls_disable   = "true"
  tls_cert_file = "vault.crt"
  tls_key_file  = "vault.key"
}

backend "raft" {
  path    = "/opt/vault/vault-data"
  node_id = "testcluster-one"
}

log_level = "trace"
log_format = "json"

EOFC
```

5. Start the Vault server
```
vault server -config  /opt/vault/vault-server.hcl 1> /opt/vault/vault-log/vault.log 2>&1 &
```

6. Init the Vault server and preserve the Unseal and Root Tokens
```
alias vault='VAULT_ADDR=http://127.0.0.1:8200 vault'
vault operator init > /root/.vault_init.txt
```

7. Validate the Vault server status and sealing information
```shell
vault status
```

8. Unseal Vault server using the keys generated at init phase (step 6.)
```shell
for i in $(seq 3) ; do
   t_var=$(grep "Key ${i}" /root/.vault_init.txt|awk '{print $NF}')
   echo -ne "\nUnsealing with Key $i having value $t_var" && \
   vault operator unseal $t_var 1>/dev/null && \
   echo -ne " - [OK]" || echo -ne " - [FAILED]"
done
echo -ne "\nUnsealing - [Done] \n" ; \
vault status |awk '{if ($1=="Sealed") {print $0 "   <=== OBSERVE the Sealed value!"} else {print  $0}}'
```

9. Observe the file structure tree
```plaintext
root@testubuntu: /opt/vault# tree
.
|-- vault-data
|   |-- raft
|   |   |-- raft.db
|   |   `-- snapshots
|   `-- vault.db
|-- vault-log
|   `-- vault.log
`-- vault-server.hcl
```

10. Perform a login using the Root Token save in the 
```shell
vault login $(grep "^Initial Root" /root/.vault_init.txt|awk '{print $NF}')
```

11. Observe the enabled secrets and perform a seal operation of the Vault server
```shell
vault secrets list
vault operator seal
```

12. Observe the Sealed value and also that listing the enabled secrets is failing ( * Vault is sealed ) 
```shell
vault status
vault secrets list
```
