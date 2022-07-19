
# Minimal install, configure and unseal a Vault Server running on Ubuntu (generic x64 bit)
----
The scope of this repository is to provide the steps for install, configure and unseal a Vault server
enerate-root-tokens-using-unseal-keys

----



1. Creating directory structure
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
unzip vault*.zip && cp vault /usr/bin/vault
```

4. Create the configuration for Vault using HCL format file /opt/vault/vault-server.hcl 
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

7. Validate the Vault status
```shell
vault status
```

8. Unseal Vault Server using the keys generated at init phase
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
