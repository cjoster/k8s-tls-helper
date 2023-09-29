# TLS tools

## Overview

These tools are attempted 

## Installation

You can use the `./install.sh` scrip to install these tools. It copies two
scripts into one of several appropriate `bin1` directories.

## Usage

How to use this software:

### Create an empty directory.

```bash
mkdir cert
cd cert
```

### Generate a CSR

```bash
createcsr
```

You will be prompted for the primary FQDN of the host. An OpenSSL configuration file
will be opened in `$EDITOR` or `vim` if `$EDITOR` is not set. Use `:cq` in `vim`
to abort creation of the CSR.

Upon save and exit, the CSR will be created, written to disk, and displayed on the screen.

You can run `createcsr` repeatedly. Each time it will reuse the key, open the configuration
file in your editory, and generate a CSR.

### Creating test PKI

There is a test PKI in the `testCA` directory. You can stand it up by cd'ing into that director
and running the `./create` script in that directory.

```bash
cd ../testCA
./create
```

### Signing your CSR

Change back into your cert directory.

```bash
cd ../cert
```

Sign your CSR

```
openssl x509 -req -in *.csr -extfile *.cnf -extensions \
	req_ext -CA ../testCA/intermediate-2.crt \
	-CAkey ../testCA/intermediate-2.key -set_serial 3 \
	-days 365 -text | tee > YOUR-FQDN.crt
```

### Put the intermediate and root files together

Create the `root.crt` and `intermediate.crt` files.

```bash
cat ../testCA/root.crt > root.crt
cat ../testCA/intermediate-2.crt ../testCA/intermediate-1.crt > intermediate.crt
```

### Verify your certificate

```bash
openssl verify -CAfile root.crt -untrusted intermediate.crt YOUR-FQDN.crt
```

### Make the k8s secret

```bash
makesecret example-tls-secret my-namespace
```

This output can be piped to `kubectl apply -f -`.

## Configuration

### Encrypted private keys

Setting the environment variable `ENCRYPT_PRIVATE_KEY` will cause `createcsr`
to create encrypted private keys. You will be prompted for the passphrase
during creationt.

This also means that the passphrase must be entered each time you perform
operations on the CSR or secret.

### Names of intermediate and root CA files

At the top of `makesecret` are two configuration variables:

```bash
intermediate_file="intermediate.crt"
root_ca_file="root.crt"
```

These can be tuned to your preferences. The intermediates and root CA files are
seperated from the certificate so that they can be symlinked.

### OpenSSL Configuration

There are a handful of tunables in these scripts. There's an inlins OpenSSL config file
in `makecsr` that can/should be tuned to your organization so that fewer edits are needed
for each certificate request.

The inline config file looks like this.

```bash
cat > "${fqdn}.cnf" << EOF
[ req ]
distinguished_name             = req_dn
default_md                     = sha256
req_extensions                 = req_ext

[ req_dn ]
countryName                    = Country Name (2 letter code)
stateOrProvinceName            = State or Province Name
localityName                   = Locality Name
0.organizationName             = Organization Name
organizationalUnitName         = Organizational Unit Name
# commonName has been deprecated for more than 10 years. Please don't use it.
# commonName                    = Common Name (e.g. your hostname)

countryName_default            = US
stateOrProvinceName_default    = California
localityName_default           = Palo Alto
0.organizationName_default     = VMware, Inc
organizationalUnitName_default = Tanzu Labs
# commonName_default            = ${fqdn}

[ req_ext ]
basicConstraints               = critical, CA:false
# Other useful values for keyUsage for a leaf certificate is keyEncipherment,
# which is not useful for sign-only key algorithms like ECC.
keyUsage                       = critical, digitalSignature
# Other useful values for extendedKeyUsage is
#   clientAuth:  Used for client authentication. Can be combined with serverAuth
#                for use-cases where the same certificate is used as both a server
#                and a client (like, say, mail servers that mTLS authenticate with
#                each other)
#   codeSigning: For signing binaries
extendedKeyUsage               = serverAuth
subjectKeyIdentifier           = hash
subjectAltName                 = @alt_names

[ alt_names ]
DNS.1 = ${fqdn}
DNS.2 = $(echo "${fqdn}" | sed 's/\..*//')
# IP addresses are frowned upon in certs, but some like to do it
# the problem is that they put them in as DNS entries
# IP.1 = 1.2.3.4
EOF
```

Be mindful of the `cat` command at the top and the `EOF` at the end.
