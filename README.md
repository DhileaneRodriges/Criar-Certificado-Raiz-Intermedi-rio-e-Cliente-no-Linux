# Criar-Certificado-Raiz-Intermedi-rio-e-Cliente-no-Linux
Este guia explica como criar um certificado SSL autoassinado, incluindo uma Autoridade Certificadora (CA) raiz, uma CA intermediária e certificados de servidor e cliente assinados pela CA intermediária.  
A seguir, as etapas para criação e verificação dos certificados criados.

## 1. Crie a Estrutura de Diretórios e Arquivos
```bash
# Pasta base
mkdir ca_selfsign
cd ca_selfsign

# CA Raiz: cria pastas e arquivos
mkdir certs crl newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial

# CA Intermediaria: cria pastas e arquivos
mkdir -p intermediate/certs intermediate/crl intermediate/csr intermediate/newcerts intermediate/private
chmod 700 intermediate/private
touch intermediate/index.txt
echo 1000 > intermediate/serial
echo 1000 > intermediate/crlnumber
```

## 2. Criar o Arquivo de Configuração openssl.cnf para a CA Raiz
```bash
cat > openssl.cnf << 'EOF'
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = .
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem
default_md        = sha256
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 7300
preserve          = no
policy            = policy_strict
unique_subject    = no

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits        = 4096
default_md          = sha256
string_mask         = utf8only
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = GB
stateOrProvinceName             = State or Province Name
stateOrProvinceName_default     = England
localityName                    = Locality Name
localityName_default            = Cambridge
0.organizationName              = Organization Name
0.organizationName_default      = University of Cambridge
organizationalUnitName          = Organizational Unit Name
organizationalUnitName_default  = Computer Laboratory
commonName                      = Common Name
commonName_default              = TLS root CA CAMB
commonName_max                  = 64
emailAddress                    = Email Address
emailAddress_default            = carlos.molina@cl.cam.ac.uk
emailAddress_max                = 64

[ v3_ca ]
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always,issuer
basicConstraints        = critical, CA:true
keyUsage                = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always,issuer
basicConstraints        = critical, CA:true, pathlen:0
keyUsage                = critical, digitalSignature, cRLSign, keyCertSign
EOF
```

## 3. Criar a Chave Privada e o Certificado Autoassinado da CA Raiz
```bash
# Gerar a chave privada da CA raiz
openssl genrsa -aes256 -out private/ca.key.pem 4096
chmod 400 private/ca.key.pem

# Gerar o certificado autoassinado da CA raiz
openssl req -config openssl.cnf \
      -key private/ca.key.pem \
      -new -x509 -days 7300 -sha256 -extensions v3_ca \
      -out certs/ca.cert.pem
chmod 444 certs/ca.cert.pem
```

## 4. Criar o Arquivo de Configuração openssl.cnf para a CA Intermediária
```bash
cat > intermediate/openssl.cnf << 'EOF'
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = ./intermediate
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
crlnumber         = $dir/crlnumber
crl               = $dir/crl/intermediate.crl.pem
private_key       = $dir/private/intermediate.key.pem
certificate       = $dir/certs/intermediate.cert.pem
default_md        = sha256
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_loose
unique_subject    = no

[ policy_loose ]
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits        = 2048
default_md          = sha256
string_mask         = utf8only
distinguished_name  = req_distinguished_name

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = GB
stateOrProvinceName             = State or Province Name
stateOrProvinceName_default     = England
localityName                    = Locality Name
localityName_default            = Cambridge
0.organizationName              = Organization Name
0.organizationName_default      = University of Cambridge
organizationalUnitName          = Organizational Unit Name
organizationalUnitName_default  = Computer Laboratory
commonName                      = Common Name
commonName_default              = TLS server CAMB
commonName_max                  = 64
emailAddress                    = Email Address
emailAddress_default            = carlos.molina@cl.cam.ac.uk
emailAddress_max                = 64

[ v3_ca ]
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always,issuer
basicConstraints        = critical, CA:true, pathlen:0
keyUsage                = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always,issuer
basicConstraints        = critical, CA:true, pathlen:0
keyUsage                = critical, digitalSignature, cRLSign, keyCertSign

[ server_cert ]
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid,issuer
basicConstraints        = CA:FALSE
keyUsage                = critical, digitalSignature, keyEncipherment
extendedKeyUsage        = serverAuth
nsCertType              = server
nsComment               = "OpenSSL Generated Server Certificate"

[ usr_cert ]
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid,issuer
basicConstraints        = CA:FALSE
keyUsage                = critical, digitalSignature, keyEncipherment
extendedKeyUsage        = clientAuth, emailProtection
nsCertType              = client, email
nsComment               = "OpenSSL Generated Client Certificate"

[ crl_ext ]
authorityKeyIdentifier  = keyid:always

[ ocsp ]
basicConstraints        = CA:FALSE
keyUsage                = critical, digitalSignature
extendedKeyUsage        = critical, OCSPSigning
EOF
```

## 5. Criar a Chave Privada e o CSR da CA Intermediária
```bash
# Gerar a chave privada da CA intermediária
openssl genrsa -aes256 -out intermediate/private/intermediate.key.pem 4096
chmod 400 intermediate/private/intermediate.key.pem

# Gerar o CSR da CA intermediária
openssl req -config intermediate/openssl.cnf -new -sha256 \
      -key intermediate/private/intermediate.key.pem \
      -out intermediate/csr/intermediate.csr.pem
```

## 6. Assinar o Certificado da CA Intermediária com a CA Raiz
```bash
openssl ca -config openssl.cnf -extensions v3_intermediate_ca \
      -days 3650 -notext -md sha256 \
      -in intermediate/csr/intermediate.csr.pem \
      -out intermediate/certs/intermediate.cert.pem
chmod 444 intermediate/certs/intermediate.cert.pem
```

## 7. Criar a Cadeia de Certificados da CA
```bash
cat intermediate/certs/intermediate.cert.pem \
      certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
chmod 444 intermediate/certs/ca-chain.cert.pem
```

## 8. Criar a Chave Privada e o Certificado do Servidor
```bash
# Gerar a chave privada do servidor
openssl genrsa -aes256 -out intermediate/private/server.key.pem 2048
chmod 400 intermediate/private/server.key.pem

# Criar o CSR do servidor
openssl req -config intermediate/openssl.cnf \
      -key intermediate/private/server.key.pem \
      -new -sha256 -out intermediate/csr/server.csr.pem

# Assinar o certificado do servidor com a CA intermediária
openssl ca -config intermediate/openssl.cnf -extensions server_cert \
      -days 375 -notext -md sha256 \
      -in intermediate/csr/server.csr.pem \
      -out intermediate/certs/server.cert.pem
chmod 444 intermediate/certs/server.cert.pem

# Criar a cadeia de certificados do servidor
cat intermediate/certs/server.cert.pem \
      intermediate/certs/ca-chain.cert.pem > intermediate/certs/server.chain.pem
chmod 444 intermediate/certs/server.chain.pem
```

## 9. Criar a Chave Privada e o Certificado do Cliente
```bash
# Gerar a chave privada do cliente
openssl genrsa -aes256 -out intermediate/private/client.key.pem 2048
chmod 400 intermediate/private/client.key.pem

# Criar o CSR do cliente
openssl req -config intermediate/openssl.cnf \
      -key intermediate/private/client.key.pem \
      -new -sha256 -out intermediate/csr/client.csr.pem

# Assinar o certificado do cliente com a CA intermediária
openssl ca -config intermediate/openssl.cnf -extensions usr_cert \
      -days 375 -notext -md sha256 \
      -in intermediate/csr/client.csr.pem \
      -out intermediate/certs/client.cert.pem
chmod 444 intermediate/certs/client.cert.pem

# Criar a cadeia de certificados do cliente
cat intermediate/certs/client.cert.pem \
      intermediate/certs/ca-chain.cert.pem > intermediate/certs/client.chain.pem
chmod 444 intermediate/certs/client.chain.pem
```

## Verificar os certificados
```bash
# Verificar o certificado do servidor
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem \
      intermediate/certs/server.cert.pem

# Verificar o certificado do cliente
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem \
      intermediate/certs/client.cert.pem
```
