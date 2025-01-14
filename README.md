# TMail LSC plugin

This a plugin for LSC, using TMail REST API


### Goal

The object of this plugin is to synchronize addresses aliases and users from one referential to a [TMail server](https://tmail.linagora.com/).

### Address Aliases Synchronization

For example, it can be used to synchronize the aliases stored in the LDAP of an OBM instance to the TMail Server(s) of a TMail deployment.

#### Architecture

Given the following LDAP entry:
```
dn: uid=rkowalsky,ou=users,dc=linagora.com,dc=lng
[...]
mail: rkowalsky@linagora.com
mailAlias: remy.kowalsky@linagora.com
mailAlias: remy@linagora.com
```

This will be represented as the following TMail address alias:
```bash
$ curl -XGET http://ip:port/address/aliases/rkowalsky@linagora.com

[
  {"source":"remy.kowalsky@linagora.com"},
  {"source":"remy@linagora.com"}
]
```

As addresses aliases in TMail are only created if there are some sources, an LDAP entry without mailAlias attribute won't be synchronized.

The pivot used for the synchronization in the LSC connector is the email address, here `rkowalsky@linagora.com` stored in the `email` attribute.

The destination attribute for the LSC aliases connector is named `sources`.

### Users Synchronization

For example, it can be used to synchronize the users stored in the LDAP of an OBM instance to the TMail Server(s) of a TMail deployment.

#### Architecture
Given the following LDAP entries:

```
dn: uid=james-user, ou=people, dc=james,dc=org
mail: james-user@james.org
[...]

dn: uid=james-user2, ou=people, dc=james,dc=org
mail: james-user2@james.org
[...]

dn: uid=james-user3, ou=people, dc=james,dc=org
mail: james-user3@james.org
[...]
```

This will be represented as the following TMail users:

```bash
$ curl -XGET http://ip:port/users

[
  {"username":"james-user2@james.org"},
  {"username":"james-user4@james.org"}
]
```

If LDAP entry with the `mail` attribute exists but not synchronized, the user will be created with choose:
- Generating random password
- [Synchronizing existing password](https://github.com/lsc-project/lsc-james-plugin/issues/2)

If LDAP entry has no `mail` attribute corresponding, the user will be deleted.

Expected Result:

    - james-user@james.org -> create
    - james-user2@james.org -> nothing happens
    - james-user3@james.org -> create
    - james-user4@james.org -> delete

```bash 
$ curl -XGET http://ip:port/users

[
  {"username":"james-user@james.org"},
  {"username":"james-user2@james.org"},
  {"username":"james-user3@james.org"}
]
```

The pivot used for the synchronization in LSC connector is email address. For this case, `james-user@james.org` is stored in `email` attribute.

### Domain contact Synchronization

For example, it can be used to synchronize the domain contact stored in a LDAP instance to the TMail Server(s) of a TMail deployment in order to empower auto-complete.

#### Architecture
Given the following LDAP entries:

```
dn: uid=renecordier, ou=people, dc=james,dc=org
mail: renecordier@james.org
givenName: Rene
sn: Cordier
[...]

dn: uid=tungtranvan, ou=people, dc=james,dc=org
mail: tungtranvan@james.org
givenName: Tung
sn: Tran Van
[...]
```

This will be represented as the following TMail domain contacts:

```bash
$ curl -XGET http://ip:port/domains/contacts

["renecordier@james.org", "tungtranvan@james.org"]
```

Second contact (tungtranvan@james.org) details:
```bash
$ curl -XGET http://ip:port/domains/james.org/contacts/tungtranvan

{
    "id": "2",
    "emailAddress": "tungtranvan@james.org",
    "firstname": "Tung",
    "surname": "Tran Van"
}
```

LDAP entries's `givenName` and `sn` are Optional.

The pivot used for the synchronization in the LSC connector is the email address, here `tungtranvan@james.org` stored in the `email` attribute.

The destination attributes for the LSC aliases connector are named `firstname` and `surname`.

### Configuration

The plugin connection needs a JWT token to connect to TMail. To configure this JWT token, set the `password` field of the plugin connection as the JWT token you want to use.

The `url` field of the plugin connection must be set to the URL of TMail' webadmin.

The `username` field of the plugin is ignored for now.

### Usage

There is an example of configuration in the `sample` directory. The `lsc.xml` file describe a synchronization from an OBM LDAP to a TMail server.
The values to configure are:
- `connections.ldapConnection.url`: The URL to the LDAP of OBM
- `connections.ldapConnection.username`: An LDAP user which is able to read the OBM aliases
- `connections.ldapConnection.password`: The password of this user

- `connections.pluginConnection.url`: The URL to the TMail Webadmin
- `connections.pluginConnection.password`: the JWT token used to connect the TMail Webadmin, it must includes an admin claim.

- `tasks.task.ldapSourceService.baseDn`: The search base of the users to synchronize.


The domains used in the aliases must have been previously created in TMail.
Otherwise, if a user have a single alias pointing to an unknown domain, none of her aliases will be added.

For the domain synchronization, you can specify the wished domain list to be synchronized by specify the dedicated ENV variable with key `DOMAIN_LIST_TO_SYNCHRONIZE` and DELIMITER `,`.
If you omit this environment variable setting, all domains contact will be synchronized from LDAP.

The jar of the TMail LSC plugin (`target/lsc-tmail-plugin-1.0-distribution.jar`) must be copied in the `lib` directory of your LSC installation.
Then you can launch it with the following command line:

```bash
JAVA_OPTS="-DLSC.PLUGINS.PACKAGEPATH=org.lsc.plugins.connectors.james.generated" bin/lsc --config /home/rkowalski/Documents/lsc-james-plugin/sample/ldap-to-james/ --synchronize all --clean all --threads 1
```

If don't want to delete dangling data, run this command without `--clean all` parameter.

### Packaging

WIP