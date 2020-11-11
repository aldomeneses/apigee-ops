# Integracion Okta y Apigee

- [Integracion Okta y Apigee](#integracion-okta-y-apigee)
  - [Prerequisitos](#prerequisitos)
  - [Configuracion de SAML IDP](#configuracion-de-saml-idp)
  - [Instalacion de Apigee SSO](#instalacion-de-apigee-sso)
    - [Creacion de llaves publicas y privadas](#creacion-de-llaves-publicas-y-privadas)
    - [Creacion de `self-signed.cert`](#creacion-de-self-signedcert)
    - [Instalar modulo apigee-sso](#instalar-modulo-apigee-sso)
    - [Instalacion de script](#instalacion-de-script)
    - [Habilitar SSO en Edge-UI](#habilitar-sso-en-edge-ui)
  - [Activacion de Sign Out](#activacion-de-sign-out)
  - [Throubleshooting](#throubleshooting)

----

## Prerequisitos

1.-Contar con cuenta en okta.com

2.-Tener permisos de administrador


----
## Configuracion de SAML IDP

Dentro de okta.com es necesario crear una nueva "Application" la cual debe tener los siguientes campos configurados:

Single sign on URL: http://`hostname`:9099/saml/SSO/alias/apigee-saml-login-opdk

Audience URI (SP Entity ID) : apigee-saml-login-opdk

Name ID format: EmailAddress

En la seccion "ATTRIBUTE STATEMENTS (OPTIONAL)" es necesario tener configurado:
| Name     | Value          |
|----------|----------------|
| FistName | user.firstName |
| LastName | user.lastName  |
| Email    | user.email     |


----

## Instalacion de Apigee SSO
Antes de realizar la instalacion de `apigee-sso` es necesario crear los certificados necesarios, a continuacion se listan los comandos a seguir:

**Nota:** los comandos pueden ser ejecutados como `sudo`

### Creacion de llaves publicas y privadas
Para ello es necesario crear el directorio donde estaran contenidos los archivos `privkey.pem  pubkey.pem` mismos que deberan ser propiedad del usuario `apigee`.
```
sudo mkdir -p /opt/apigee/customer/application/apigee-sso/jwt-keys
sudo cd /opt/apigee/customer/application/apigee-sso/jwt-keys/
sudo openssl genrsa -out privkey.pem 2048
sudo openssl rsa -pubout -in privkey.pem -out pubkey.pem
sudo chown apigee:apigee *.pem
```
### Creacion de `self-signed.cert`
Para crear el archivo `self-signed.cert`, es neceario crearlo en otro directorio, asi como antes de ello crear los archivos `server.key` y `server.csr`

```
sudo mkdir -p /opt/apigee/customer/application/apigee-sso/saml/
sudo cd /opt/apigee/customer/application/apigee-sso/saml/
sudo openssl genrsa -aes256 -out server.key 1024 
```
**Nota:** Pedira un password

Eliminar passphrase de llave
```
sudo openssl rsa -in server.key -out server.key
```
Generacion del certificado firmado por CA

```
sudo openssl req -x509 -sha256 -new -key server.key -out server.csr
```

Generacion del certificado con 365 para expirar y asignacion de propietario apigee
```
sudo openssl x509 -sha256 -days 365 -in server.csr -signkey server.key -out selfsigned.crt
sudo chown apigee:apigee server.key
sudo chown apigee:apigee selfsigned.crt
```

### Instalar modulo apigee-sso
```
/opt/apigee/apigee-setup/bin/setup.sh -p sso -f /tmp/apigee/configFile-sso
```

**Nota:** configFile-sso es un archivo que hay que configurar

### Instalacion de script

```
/opt/apigee/apigee-service/bin/apigee-service apigee-ssoadminapi install
```

### Habilitar SSO en Edge-UI
```
/opt/apigee/apigee-service/bin/apigee-service edge-ui configure-sso -f /tmp/apigee/configFile-sso-setup
```
**Nota:** configFile-sso-setup es un archivo que hay que configurar

## Activacion de Sign Out
Para realizar la activacion de Sign Out es necesario accesar de nuevo a la aplicacion dentro de Okta, accesar a la opicion `Edit` y dentro `SAML Settings` se muestra un link `Show Advanced Settings`, dentro de Advanced Settings es neceario tener la siguiente configuracion

| Name | Value |
|------|-------|
|Enable Single Logout | &#9745; Allow application to initiate Single Logout |
|Single Logout URL:| http://`hostname`:9099/saml/SingleLogout/alias/apigee-saml-login-opdk |
|SP Issuer: | apigee-saml-login-opdk |
|Signature Certificate: | selfsigned.crt |

## Throubleshooting

Si no se ejecutan correctamente los comandos: `/opt/apigee/apigee-setup/bin/setup.sh -p sso -f` o `/opt/apigee/apigee-service/bin/apigee-service edge-ui configure-sso -f` se puede hacer rollback de una manera limpia de la siguiente manera:

1.- Detener el `apigee-sso` con el siguiente comando:
```
/opt/apigee/apigee-service/bin/apigee-service apigee-sso stop
```
2.- Eliminar la Base de Datos de `apigee-sso` con el siguiente comando:
```
psql -U postgres_username -p postgres_port -h postgres_host -c "drop database \"apigee_sso\""
```