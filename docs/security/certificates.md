---
layout: default
title: TLS Certificates
parent: Security
nav_order: 2
---

# TLS certificates

Out of the box, the Security plugin contains demo certificates so that you can quickly get started with Open Distro for Elasticsearch. Before deploying to a production environment, however, you should replace them with your own certificates.


## Generate certificates

Generate new certificates using the Security plugin [TLS tool](https://github.com/opendistro/security-tlstool).

The TLS Tool is platform independent and can be used for

* Generating root and intermediate certificate authorities (CAs)
* Generating node, client, and admin certificates
* Generating certificate signing requests (CSRs)
* Validating certificates

Beyond the actual certificates, the tool also generates configuration snippets that you can copy and paste into `elasticsearch.yml`.


### Usage

The TLS tool reads the node and certificate configuration settings from a YAML file and outputs the generated files in a configurable directory.

You can choose to create the root CA and (optional) intermediate CAs with your node certificates in one operation or separately.

1. Download the archive and unzip it.
1. Run `<install-directory>/tools/sgtlstool.sh` with the options appropriate to your use case.


#### Command line options

Name | Description
:--- | :---
-c, --config | Required. Relative or absolute path to the configuration file.
-t, --target | Relative or absolute path to the output directory. Default is `out`.
-v, --verbose | Enable detailed output. Default is false.
-f, --force | Force certificate generation despite of validation errors. Default is false.
-o, --overwrite | Overwrite existing node-, client and admin certificates if they are already present. Default is false.
-ca, --create-ca | Create new root and intermediate CAs.
-crt, --create-cert | Create certificates using an existing or newly-created local certificate authority.


#### Example

```
./sgtlstool.sh -c ../config/tlsconfig.yml -ca -crt
```

Reads the configuration from `../config/tlsconfig.yml` and generates the configured root and intermediate CAs, and the configured node, admin and client certificates, in one operation. The generated files are written to `out`.

```
./tools/sgtlstool.sh -c ../config/tlsconfig.yml -ca
```

Reads the configuration from `../config/tlsconfig.yml` and generates the configured root and intermediate CAs only.

```
./tools/sgtlstool.sh -c ../config/tlsconfig.yml -crt
```

Reads the configuration from `../config/tlsconfig.yml` and generates node, admin, and client certificates only. The root and (optional) intermediate CA certificates and keys must be present in the output (`out`) directory, and their filenames, keys and (optional) passwords must be configured in `tlsconfig.yml`.


### Root CA

To configure the root CA for all certificates, add the following lines to your configuration file:

```
ca:
   root:
      dn: CN=root.ca.example.com,OU=CA,O=Example Com\, Inc.,DC=example,DC=com
      keysize: 2048
      pkPassword: root-ca-password
      validityDays: 3650
      file: root-ca.pem
```

Option | Description
:--- | :---
dn | The complete Distinguished Name (DN) of the root CA. If you have special characters in the DN (for example, a comma), you must escape it using a `\`. Required.
keysize | The size of the private key. Default is 2048.
pkPassword | The password for the private key. Default is `auto`.
file | The file name of the certificate. Optional. Default is `root-ca`.

`pkPassword` can be one of:

- **none**: The generated private key will be unencrypted.
- **auto**: Generates a random password automatically. After the certificates have been generated, you can find the password in `root-ca.readme`. In order to use these new passwords at a later time, you must edit the TLS tool config file and set the generated passwords there.
- **other values**: Values other than `none` or `auto` are used as the password.


## Intermediate CA

In addition to the root CA you optionally also specify an intermediate CA. If an intermediate CA is configured, then the node, admin and client certificates will be signed by the intermediate CA. If you do want to use an intermediate CA, remove the following section from the configuration. The certificates are then signed by the root CA directly.

```
ca:
   intermediate:
      dn: CN=signing.ca.example.com,OU=CA,O=Example Com\, Inc.,DC=example,DC=com
      keysize: 2048
      validityDays: 3650  
      pkPassword: intermediate-ca-password
      file: intermediate-ca.pem
```


## Node and Client certificates

### Global and default settings

The default settings are applied to all generated certificates and configuration snippets. All values are optional.

```
defaults:
  validityDays: 730
  pkPassword: auto
  generatedPasswordLength: 12
  nodesDn:
    - "CN=*.example.com,OU=Ops,O=Example Com\\, Inc.,DC=example,DC=com"
  nodeOid: "1.2.3.4.5.5"
  httpEnabled: true
  reuseTransportCertificatesForHttp: false
```

Options:

Name | Description
:--- | :---
validityDays | Validity of the generated certificates, in days. Default: 730. Can be overwritten for each certificate separately.
pkPassword | Password of the private key. Default: auto. Can be overwritten for each certificate separately.
generatedPasswordLength | Length of the auto-generated password for the private keys. Only takes effect when `pkPassword` is set to `auto`. Default: 12. Can be overwritten for each certificate separately.
nodesDn | Value of the `opendistro_security.nodes_dn` in the configuration snippet. Optional. If omitted, all DNs of all node certificates are listed (default). If you want to define your node certificates using wildcards or regular expressions, set the values here.
nodeOid | If you want to use OIDs to mark legitimate node certificates instead of listing them in `opendistro_security.nodes_dn`, set the OID here. It will be included in the SAN section of all node certificates. The default doesn't add the OID to the SAN section.
httpsEnabled | Whether to enable TLS on the REST layer or not. Default is true.
reuseTransportCertificatesForHttp | If set to false, individual certificates for REST and Transport are generated. If set to true, the node certificates are also used on the REST layer. Default is false.
verifyHostnames | Set this to true to enable hostname verification. Default is false.
resolveHostnames | Set this to true to resolve hostnames against DNS. Default is false.


### Node certificates

To generate node certificates, add the node name, the Distinguished Name (DN), the hostname(s), and/or the IP address(es) in the `nodes` section:

```
nodes:
  - name: node1
    dn: CN=node1.example.com,OU=Ops,O=Example Com\, Inc.,DC=example,DC=com
    dns: node1.example.com
    ip: 10.0.2.1
  - name: node2
    dn: CN=node2.example.com,OU=Ops,O=Example Com\, Inc.,DC=example,DC=com
    dns:
      - node2.example.com
      - es2.example.com
    ip:
      - 10.0.2.1
      - 192.168.2.1
  - name: node3
    dn: CN=node3.example.com,OU=Ops,O=Example Com\, Inc.,DC=example,DC=com
    dns: node3.example.com
```    

#### Options

Name | Description
|---|---
name | Name of the node, will become part of the filenames. Required.
dn | The Distinguished Name of the certificate. If you have special characters in the DN, you need to quote them correctly. Required.
dns | The hostname(s) this certificate is valid for. Should match the `hostname` of the node. Optional, but recommended.
ip | The IP(s) this certificate is valid for. Optional. Hostnames are the better solution.


## Admin and client certificates

To generate admin and client certificates, add the following lines to the configuration file:

```
clients:
  - name: spock
    dn: CN=spock.example.com,OU=Ops,O=Example Com\, Inc.,DC=example,DC=com
  - name: kirk
    dn: CN=kirk.example.com,OU=Ops,O=Example Com\, Inc.,DC=example,DC=com
    admin: true
```

Options:

Name | Description
:--- | :---
name | Name of the certificate, will become part of the file name
dn | The complete Distinguished Name of the certificate. If you have special characters in the DN, you need to quote them correctly.
admin  | If set to true, this certificate is marked as admin certificate in the generated configuration snippet.

Note that you need to mark at least one client certificate as admin certificate.


## Adding certificates after the first run

You can always add more node- or admin certificates as you need them after the initial run of the tool. As a precondition

* the root CA and, if used, the intermediate certificates and keys must be present in the output folder
* the password of the root CA and, if used, the intermediate CA must be present in the config file

If you use auto-generated passwords, copy them from the generated `root-ca.readme` file to the configuration file.

Certificates that have already been generated in a previous run of the tool will be left untouched unless you run the tool with the `-o,--overwrite` switch. In this case existing files are overwritten. If you have chosen to auto-generate passwords, new keys with auto-generated passwords are created.

## Creating CSRs

If you just want to create CSRs to submit them to your local CA, you can omit the `ca` part of the config complete. Just define the `default`, `node` and `client` section, and run the TLS tool with the `-csr` switch:

```
/sgtlstool.sh -c ../config/example.yml -csr
```


## Validating certificates

The TLS diagnose tool can be used to verify your certificates and your certificate chains:

```
<installation directory>/tools/sgtlsdiag.sh
<installation directory>/tools/sgtlsdiag.bat
```

### Command line options

Name | Description
:--- | :---
-ca,--trusted-ca | Path to a PEM file containing the certificate of a trusted CA
-crt,--certificates | Path to PEM files containing certificates to be checked
-es,--es-config | Path to the ElasticSearch config file containing the Security Plugin TLS configuration
-v,--verbose | Enable detailed output

### Example: Checking PEM certificates directly

```
<installation directory>/tools/sgtlsdiag.sh -ca out/root-ca.pem -crt out/node1.pem
```

The diagnose tool will output the contents of the provided certificates and checks that the trust chain is valid. Example:

```
------------------------------------------------------------------------
Certificate 1
------------------------------------------------------------------------
            SHA1 FPR: 6A2049B9C7F8E48A79B81D60378F4C98EFD34B75
             MD5 FPR: 72C7F44B875FA777C6A4F047060304C6
Subject DN [RFC2253]: CN=node1.example.com,OU=Ops,O=Example Com\, Inc.,DC=example,DC=com
       Serial Number: 1519568340695
 Issuer DN [RFC2253]: CN=signing.ca.example.com,OU=CA,O=Example Com\, Inc.,DC=example,DC=com
          Not Before: Sun Feb 25 15:19:02 CET 2018
           Not After: Wed Feb 23 15:19:02 CET 2028
           Key Usage: digitalSignature nonRepudiation keyEncipherment
 Signature Algorithm: SHA256WITHRSA
             Version: 3
  Extended Key Usage: id_kp_serverAuth id_kp_clientAuth
  Basic Constraints: -1
                SAN:
                  dNSName: node1.example.com
                  iPAddress: 10.0.2.1

------------------------------------------------------------------------
Certificate 2
------------------------------------------------------------------------
            SHA1 FPR: BBCF41A85E85385B8301FA641E9BF002086AAED5
             MD5 FPR: 3E1C2881A05FF08B4582F3AC2D2443B9
Subject DN [RFC2253]: CN=signing.ca.example.com,OU=CA,O=Example Com\, Inc.,DC=example,DC=com
       Serial Number: 2
 Issuer DN [RFC2253]: CN=root.ca.example.com,OU=CA,O=Example Com\, Inc.,DC=example,DC=com
          Not Before: Sun Feb 25 15:19:02 CET 2018
           Not After: Wed Feb 23 15:19:02 CET 2028
           Key Usage: digitalSignature keyCertSign cRLSign
 Signature Algorithm: SHA256WITHRSA
             Version: 3
  Extended Key Usage: null
  Basic Constraints: 0
                SAN: (none)
------------------------------------------------------------------------
Trust anchor:
DC=com,DC=example,O=Example Com\, Inc.,OU=CA,CN=root.ca.example.com
```


## Production configuration

The Security plugin supports certificates in the following formats:

* X509 PEM certificates and PKCS8 private keys
* Keystores and truststores in JKS or PKCS12/PFX format


### Types of certificates

The Security plugin distinguishes between the following types of certificates

- Node certificates
- Client certificates
- Admin certificates

**Node certificates** are used to identify and secure traffic between Elasticsearch nodes on the transport layer (inter-node traffic). For this kind of traffic, no permission checks are applied, i.e. every request is allowed. Therefore, these certificates must meet some special requirements. See below for details and options.

**Client certificates** are regular TLS certificates without any special requirements. They are used to identify Elasticsearch clients on the REST and transport layer. They can be used for HTTP client certificate authentication or when using a Java Transport Client on transport layer.

**Admin certificates** are **client certificates** that have elevated rights to perform administrative tasks. You need an admin certificate to change the Security plugin configuration via the [sgadmin](sgadmin.md) command line tool, or to use the [REST management API](restapi_api_access.md). Admin certificates are configured in `elasticsearch.yml` by simply stating their DN(s). You can use any valid client certificate as an admin certificate.  

Do not use a node certificate as admin certificate! Node and admin certificates must always be kept separate.
{: .note }


## Node certificates

The Security plugin needs to securely and reliably identify internal communication between Elasticsearch nodes (inter-node traffic). This communication happens for example if one node receives a GET request on the HTTP layer, but needs to forward it to another node that holds the actual data.

The Security plugin offers several ways of identifying inter-node traffic. Depending on your PKI setup and capabilities, you can choose from one of the following methods.


### Listing DNs of node certificates

The simplest way of configuring node certificates is to list the DNs of these certificates in `elasticsearch.yml`. Wildcards and regular expressions are supported:

```yaml
opendistro_security.nodes_dn:
  - 'CN=node.other.com,OU=SSL,O=Test,L=Test,C=DE'
  - 'CN=*.example.com,OU=SSL,O=Test,L=Test,C=DE'
  - 'CN=elk-devcluster*'
  - '/CN=.*regex/'
```

All certificate DNs listed here are considered valid node certificates.


### Using an OID value as SAN entry

This is the default setting of the Security plugin when no `opendistro_security.nodes_dn` is specified. It is also used as fallback if the certificate's DN does not match any DN configured in `opendistro_security.nodes_dn`.

`OID` stands for an object identifier and in this context it is used to identify an X.509 certificate extension, i.e. an additional data field stored in the certificate, which is not predefined by the standard.

The `OID` is defined in the `Subject Alternative Name (SAN)` section of the certificate, and must have the default value `1.2.3.4.5.5` for node certificates. If this value is found, the certificate is considered to be a valid node certificate.

The OID approach is more flexible than listing the DNs individually: All certificates containing this OID value in the SAN field are considered valid node certificates, so you can add new nodes on the fly without changing the `opendistro_security.nodes_dn` setting.

If you want to use this approach, make sure that your CSR includes `oid:1.2.3.4.5.5` in the `SAN` field.


#### Using a custom OID value

If you cannot set the `OID` to the default value `1.2.3.4.5.5`, but you are able to use a different value, you can configure this value in `elasticsearch.yml` by setting:

```
opendistro_security.cert.oid: <String>
```


### Custom implementation

You can also provide your own implementation to identify inter-node traffic. Please see chapter [Expert features](tls_expert.md) for details.


## Extended key usage settings

A node certificate is used for server authentication and client authentication:

* If a node issues an request to another node, it acts like a client, requesting data from a server.

* If a node receives requests from another node, it acts like a server, accepting requests from the client node.

Therefore, for node certificates the extended key usage settings must include both `serverAuth` and `clientAuth`:

```bash
#3: ObjectId: 2.5.29.37 Criticality=false
ExtendedKeyUsages [
  serverAuth
  clientAuth
]
```


## Special characters and whitespaces

The Security plugin uses the [String Representation of Distinguished Names (RFC1779)](https://www.ietf.org/rfc/rfc1779.txt) when validating node certificates.

If parts of your DN contain special characters, for example a comma, make sure it is escaped properly in your `opendistro_security.nodes_dn` configuration, for example:

```yaml
opendistro_security.nodes_dn:
  - 'CN=node.other.com,OU=SSL,O=My\, Test,L=Test,C=DE'
```

Omit whitespaces between the individual parts of the DN. Instead of:

```yaml
opendistro_security.nodes_dn:
  - 'CN=node.other.com, OU=SSL, O=MyTest, L=Test,C=DE'
```

use:

```yaml
opendistro_security.nodes_dn:
  - 'CN=node.other.com,OU=SSL,O=MyTest,L=Test,C=DE'
```


## emailAddress Fields

Using the emailAddress as part of the DN is deprecated. It is recommended to add the email as SAN entry to the certificate instead.


## Working with aliases

While it is common to have separate files for storing the trusted root certificates ("truststore") and the client certificate, technically speaking there is no reason to do so. Means, you can also store the root certificates, intermediate certificates and client certificates in the same file.

In this case you can work with aliases to specify which certificate in your keystore should be used for what purpose.

When importing certificates in your keystore, you can set an alias name like:

```bash
keytool -importcert -file cert.pem -keystore keystore.jks -alias "myalias"
```

You can later use this alias in the Security plugin TLS configuration to refer to exactly this certificate (or certificate chain), for example:

```yaml
opendistro_security.ssl.transport.keystore_filepath keystore.jks
opendistro_security.ssl.transport.keystore_alias: myalias
```


## Client and admin certificates

There are no special requirements for client and admin certificates. So any certificate signed by the Root CA or intermediate CA can be used.

Admin certificates have elevated permissions, including the permission to change the Security plugin index. To assign these elevated permissions to a certificate, simply list them in `elasticsearch.yml`:

```yaml
opendistro_security.authcz.admin_dn:
  - CN=admin,OU=SSL,O=Test,L=Test,C=DE
```

Note: For security reasons, wildcards and regular expressions are not supported for admin certificates.
{: .note }


## Disable TLS Client Renegotiation

Secure Client-Initiated Renegotiation is disabled by default, because it could be used for denial of service attacks on the HTTP port. If you want to enable Client-Initiated Renegotiation, add the following line to config/jvm.options:

```
-Djdk.tls.rejectClientInitiatedRenegotiation=false
```


## Disable TLS versions and weak ciphers

You can control the enabled ciphers and TLS protocols by the following configuration keys in `elasticsearch.yml`:

```yaml
opendistro_security.ssl.http.enabled_ciphers:
  - "TLS_DHE_RSA_WITH_AES_256_CBC_SHA"
  - "TLS_DHE_DSS_WITH_AES_128_CBC_SHA256"
opendistro_security.ssl.http.enabled_protocols:
  - "TLSv1.1"
  - "TLSv1.2"
```
