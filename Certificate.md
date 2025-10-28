# Generate CA.key
openssl genrsa -aes256 -out rootCA.key 4096

# Generate CA certificate
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.pem

# Generate Server Key
openssl genrsa -out server.key 2048

# Create san.cnf

'''
ini
[req]
default_md = sha256
req_extensions = req_ext
distinguished_name = req_distinguished_name

[req_distinguished_name]
commonName = example.com

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = example.com
DNS.2 = www.example.com
DNS.3 = api.example.com
IP.1 = 127.0.0.1
'''

# Generate server csr with san.cnf
openssl req -new -nodes -key server.key -sha256 -out server.csr -config san.cnf

# Sign csr
openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 365 -sha256 -extensions req_ext -extfile san.cnf

# Verify server certificate
openssl x509 -in server.crt -text -noout
