## FreeBSD,Wazuh and Keycloak using OpenID
## Requirements

* A keycloak service must be configure on production mode. You can install it from ports (*net/keycloak*) or packages (*pkg install keycloak*)

* A Wazuh all-in-one infrastructure must be configured like minimal for test this integration.

* On this how-to we will use *IP 172.16.0.21* for keycloak service and *IP 172.16.0.22* for Wazuh all-in-one or cluster infrastructure

## Keycloak settings

We need add somes setings to our keycloak settings

image

Create a wazuh realm

image

Create admin and all_access realm roles

image

Create a wazuh-admin user used to login to wazuh-dashboards

image

You can define what kind of actions wazuh-admin user needs before of a success login.

image

Assign admin and all_access roles to wazuh-admin user

image

Create an initial access token. Don't forget copy token because it can not be retrieved later

image

We re-use account client id from clients option. This is created automatically when wazuh realm is created. Edit account client id

image

Add admin and all_access roles to account roles

image

Modify realms roles from Client scopes/roles/Mappers

image

## Opensearch settings

Add openid settings to */usr/local/etc/opensearch/opensearch-security/config.yml* file.

```sh
openid_auth_domain:
   http_enabled: true
   transport_enabled: true
   order: 1
   http_authenticator:
     type: openid
     challenge: false
     config:
       subject_key: preferred_username
       roles_key: roles
       openid_connect_url: https://172.16.0.21:8443/realms/wazuh/.well-known/openid-configuration
       openid_connect_idp:
         enable_ssl: true
         verify_hostnames: false
         pemtrustedcas_filepath: "/usr/local/etc/opensearch/certs/server.crt.pem"
   authentication_backend:
      type: noop
```


## OpenSearch Dashboards settings

Enable OpenID authentication on opensearch-dashboards. Add the following lines to */usr/local/etc/opensearch-dashboards/opensearch_dashboards.yml* file

```sh
# Enable OpenID authentication
opensearch_security.auth.type: "openid"

# The IDP metadata endpoint
opensearch_security.openid.connect_url: "https://172.16.0.21:8443/realms/wazuh/.well-known/openid-configuration"

# Public certificate generated on production mode of keycloak service
opensearch_security.openid.root_ca: /usr/local/etc/opensearch-dashboards/certs/server.crt.pem
#opensearch_security.openid.base_redirect_url: https://172.16.0.22:5601

# The ID of the OpenID Connect client in your IdP (use account client id)
opensearch_security.openid.client_id: "account"

# The client secret of the OpenID Connect client (initial access token)
opensearch_security.openid.client_secret: "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICIzZDE1YTU5ZC0xM2I2LTQ0NzQtYWU0Ny0yOWQzYzQ4YWI3MjYifQ.eyJleHAiOjE3Mzg5NTE5MjIsImlhdCI6MTcwNzQxNTkyMiwianRpIjoiYzg0OTZjMjItNGY4ZC00OGQ4LTgxNTAtZTcyM2FmYWUzMGJkIiwiaXNzIjoiaHR0cHM6Ly8xNzIuMTYuMC4yMTo4NDQzL3JlYWxtcy93YXp1aCIsImF1ZCI6Imh0dHBzOi8vMTcyLjE2LjAuMjE6ODQ0My9yZWFsbXMvd2F6dWgiLCJ0eXAiOiJJbml0aWFsQWNjZXNzVG9rZW4ifQ.Yijrzydu17jAZEIIRj2kH5WuigTu7wfojC-CWmhUZl8"

# Disable SSL verification when using self-signed demo certificates
opensearch_security.openid.verify_hostnames: "false"

# Add roles to the scope
opensearch_security.openid.scope: "openid profile email address phone roles"
```

## Initialize OpenSearch cluster

We need apply these new changes

```sh
  # cd /usr/local/lib/opensearch/plugins/opensearch-security/tools/
  # sh -c "OPENSEARCH_JAVA_HOME=/usr/local/openjdk11 ./securityadmin.sh \
    -cd /usr/local/etc/opensearch/opensearch-security/ -cacert /usr/local/etc/opensearch/certs/root-ca.pem \
    -cert /usr/local/etc/opensearch/certs/admin.pem -key /usr/local/etc/opensearch/certs/admin-key.pem -h 172.16.0.22 -p 9200 -icl -nhnv"
```

Restarting opensearch-dashboards for applying changes

```sh
  # service opensearch-dashboards restart
```

## Testing

If all settings were applied correctly when you want access to Wazuh Dashboards *(https://172.16.0.22:5601)*, it will be redirect to Keycloak Wazuh Realm login page

image

image

## License
This project is licensed under the BSD-3-Clause license.
