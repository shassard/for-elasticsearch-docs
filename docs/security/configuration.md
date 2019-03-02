---
layout: default
title: Backend Configuration
parent: Security
nav_order: 3
---

# Backend configuration

One of the first steps to using the Security plugin is to decide on an authentication backend, which handles [steps 2-3 of the authentication flow](../concepts#authentication-flow). The plugin has an internal user database, but many people prefer to use an existing authentication backend, such as an LDAP server, or some combination of the two.

The main configuration file for authentication and authorization modules  is `plugins/opendistro_security/securityconfig/config.yml`. It defines how the Security plugin retrieves the user credentials, how it verifies these credentials, and how additional user roles are fetched from backend systems (optional).

`config.yml` has three main parts:

```yaml
searchguard:
  dynamic:
    http:
      ...
    authc:
      ...
    authz:
      ...
```

## HTTP

The `http` section has the following format:

```yaml
anonymous_auth_enabled: <true|false>
xff: # optional section
  enabled: <true|false>
  internalProxies: <string> # Regex pattern
  remoteIpHeader: <string> # Name of the header in which to look. Typically: x-forwarded-for
  proxiesHeader: <string>
  trustedProxies: <string> # Regex pattern
```

If you disable anonymous authentication, the Security plugin won't initialize if you have not provided at least one `authc`.

## Authentication

The `authc` section has the following format:

```yaml
<name>:
  http_enabled: <true|false>
  transport_enabled: <true|false>
  order: <integer>
    http_authenticator:
      ...
    authentication_backend:
      ...
```

An entry in the `authc` section is called an **authentication domain**. It specifies where to get the user credentials and against which backend they should be authenticated.

You can use more than one authentication domain. Each authentication domain has a name (e.g. `basic_auth_internal`), `enabled` flags, and an `order`. The order makes it possible to chain authentication domains together. The Security plugin uses them in the order you provide. If the user successfully authenticates with one domain, the Security plugin skips the remaining domains.

`http_authenticator` specifies which authentication method you want to use on the HTTP layer.

The syntax for defining an authenticator on the HTTP layer is:

```yaml
http_authenticator:
  type: <type>
  challenge: <true|false>
  config:
    ...
```

Allowed values for `type` are:

- basic: HTTP basic authentication. No additional configuration is needed.
- kerberos: Kerberos authentication. Additional, [Kerberos-specific configuration](#kerberos) is needed.
- jwt: JSON web token authentication. Additional, [JWT-specific configuration](#json-web-tokens) is needed.
- proxy: Use an external, proxy-based authentication. Additional, proxy-specific configuration is needed, and the `X-forwarded-for` module has to be enabled as well. See [Proxies](../proxies) for details.
- clientcert: Authentication via a client TLS certificate. This certificate must be trusted by one of the root CAs in the truststore of your nodes.

After setting an HTTP authenticator, you need to specify against which backend system you want to authenticate the user:

```yaml
authentication_backend:
  type: <type>
  config:
    ...
```

Possible vales for `type` are:

- noop: This setting means that no further authentication against any backend system is performed. Use `noop` if the HTTP authenticator has already authenticated the user completely, as in the case of JWT, Kerberos, proxy-based, or client certificate authentication.
- internal: Use the users and roles defined in `internal_users.yml` for authentication.
- ldap: Authenticate users against an LDAP server. This setting requires [additional, LDAP-specific configuration settings](#ldap).


## Authorization

After the user has been authenticated, the Security plugin can optionally collect additional user roles from backend systems. The authorization configuration has the following format:

```yaml
authz:
  <name>:
    http_enabled: <true|false>
    transport_enabled: <true|false>
    authorization_backend:
      type: <type>
      config:
        ...
```

You can define multiple entries in this section the same way as you can for authentication entries. In this case, execution order is not relevant, so there is no `order` field.

Possible vales for `type` are:

- noop: Used to skip this step altogether
- ldap: Fetch additional roles from an LDAP server. This setting requires [additional, LDAP-specific configuration settings](#ldap).


## Examples

The default `plugins/opendistro_security/securityconfig/config.yml` that ships with Open Distro for Elasticsearch contains many configuration examples. Use these examples as a starting point, and customize them to your needs.


## Kerberos

Due to the nature of Kerberos, you need to define some settings in `elasticsearch.yml` and some in `config.yml`.

In `elasticsearch.yml`, you need to define:

```yaml
opendistro_security.kerberos.krb5_filepath: '/etc/krb5.conf'
opendistro_security.kerberos.acceptor_keytab_filepath: 'eskeytab.tab'
```

`opendistro_security.kerberos.krb5_filepath` defines the path to your Kerberos configuration file. This file contains various settings regarding your Kerberos installation, for example, the realm name(s), hostname(s), and port(s) of the Kerberos key distribution center (KDC).

`opendistro_security.kerberos.acceptor_keytab_filepath` defines the path to the keytab file, which contains the principal that the Security plugin uses to issue requests against Kerberos.

`acceptor_principal: 'HTTP/localhost'` defines the principal that the Security plugin will use to issue requests against Kerberos.

The `acceptor_principal` defines the acceptor/server principal name the Security plugin uses to issue requests against Kerberos. This value must be present in the keytab file.

Due to security restrictions, the keytab file must be placed in the `<open-distro-install-dir>/conf` or a subdirectory, and the path in `elasticsearch.yml` must be relative, not absolute.
{: .warning }


## Dynamic configuration

A typical Kerberos authentication domain in `config.yml` looks like this:

```yaml
    authc:
      kerberos_auth_domain:
        enabled: true
        order: 1
        http_authenticator:
          type: kerberos
          challenge: true
          config:
            krb_debug: false
            strip_realm_from_principal: true
        authentication_backend:
          type: noop
```

Authentication against Kerberos via a browser on HTTP level is achieved using SPNEGO. Kerberos/SPNEGO implementations vary, depending on your browser/operating system. This is important when deciding if you need to set the `challenge` flag to true or false.

As with [HTTP Basic Authentication](httpbasic.md), this flag determines how the Security plugin should react when no `Authorization` header is found in the HTTP request, or if this header does not equal `negotiate`.

If set to true, the Security plugin will send a response with status code 401 and a `WWW-Authenticate` header set to `Negotiate`. This will tell the client (browser) to resend the request with the `Authorization` header set. If set to false, the Security plugin cannot extract the credentials from the request, and authentication will fail. Setting `challenge` to false thus only makes sense if the Kerberos credentials are sent in the inital request.

As the name implies, setting `krb_debug` to true will output a lot of Kerberos specific debugging messages to stdout. Use this if you encounter any problems with your Kerberos integration.

If you set `strip_realm_from_principal` to true, the Security plugin will strip the realm from the user name.

## Authentication backend

Since SPNEGO/Kerberos authenticates a user on HTTP level, no additional `authentication_backend` is needed, hence it can be set to `noop`.

## Troubleshooting

Set `krb_debug: true` in the dynamic configuration. Now any login attempt with a SPNEGO token should print diagnostic information to stdout.

If you do not see any output or use an older the Security plugin Kerberos module set the following JVM porperties manually:

* `-Dsun.security.krb5.debug=true`
* `-Djava.security.debug=gssloginconfig,logincontext,configparser,configfile`
* `-Dsun.security.spnego.debug=true`

## JSON web token

asdf

## Proxies

adsf

## LDAP

asdf
