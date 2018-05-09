+++
author = "Nick Lanng"
categories = ["vault", "hashicorp", "security"]
date = 2017-10-23T07:40:15Z
description = ""
draft = false
image = "/images/2017/10/vault.png"
slug = "getting-to-know-vault"
tags = ["vault", "hashicorp", "security"]
title = "Getting to Know Vault"

+++

#What is Vault?

Vault is a secrets management tool from the excellent people at Hashicorp. A constantly evolving solution, based on academia, that always passes security audits with flying colors. It feels like using an encrypted Redis or Memcached.

From the website - https://www.vaultproject.io/
> Vault secures, stores, and tightly controls access to tokens, passwords, certificates, API keys, and other secrets in modern computing. Vault handles leasing, key revocation, key rolling, and auditing. Through a unified API, users can access an encrypted Key/Value store and network encryption-as-a-service, or generate AWS IAM/STS credentials, SQL/NoSQL databases, X.509 certificates, SSH credentials, and more.

#Set up a development environment

I'm a Mac user with Homebrew installed, so to install Vault I simply run:
{{< highlight bash >}}
$ brew install vault
{{< /highlight >}}

For other operating systems, please refer to the binary downloads: https://www.vaultproject.io/downloads.html.

Next, I open up a terminal and start Vault in dev mode; this allows quick and easy start up for learning Vault but is **not** recommended for production deployments.

{{< highlight bash >}}
$ vault server -dev

==> Vault server configuration:

                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", tls: "disabled")
               Log Level: info
                   Mlock: supported: false, enabled: false
        Redirect Address: http://127.0.0.1:8200
                 Storage: inmem
                 Version: Vault v0.8.3
             Version Sha: 6b29fb2b7f70ed538ee2b3c057335d706b6d4e36

==> WARNING: Dev mode is enabled!

In this mode, Vault is completely in-memory and unsealed.
Vault is configured to only have a single unseal key. The root
token has already been authenticated with the CLI, so you can
immediately begin using the Vault CLI.

The only step you need to take is to set the following
environment variables:

    export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are reproduced below in case you
want to seal/unseal the Vault or play with authentication.

Unseal Key: 7bU0jT/vrbTIJ0U2LMZ7/mh7OSlF08oywbrX06qU41Q=
Root Token: 92c72f27-a303-3212-701d-795242d99ea1

==> Vault server started! Log data will stream in below:
{{< /highlight >}}

The most important part here is the environmental variable required to have the Vault command line communicate with the correct endpoint. Run the following in your terminal:

{{< highlight bash >}}
$ export VAULT_ADDR='http://127.0.0.1:8200'
{{< /highlight >}}

The Vault server is now up and running,  unsealed and ready to play with!

#What can you do with it?

All of the operations here using the `vault` command line tool are available in the API. In fact, all the `vault` command is doing is calling the API, there's no special access route for the command line. Any application with the ability to communicate with HTTP APIs can communicate with Vault.

##Key Value Storage
The most basic operation Vault provides is storing and retrieving secrets in the Key Value backend.
You can see all of the backends mounted in the server.
{{< highlight bash >}}
$ vault mounts
Path        Type       Accessor            Plugin  Default TTL  Max TTL  Force No Cache  Replication Behavior  Description
cubbyhole/  cubbyhole  cubbyhole_9c150dcf  n/a     n/a          n/a      false           local                 per-token private secret storage
secret/     kv         kv_eff8b60a         n/a     system       system   false           replicated            key/value secret storage
sys/        system     system_f08dd5fb     n/a     n/a          n/a      false           replicated            system endpoints used for control, policy and debugging
{{< /highlight >}}

In that list is a kv (Key Value) store mounted at secret/. So let's write a secret into vault.

{{< highlight bash >}}
$ vault write secret/mysecrets name=foo password=bar
Success! Data written to: secret/mysecrets
{{< /highlight >}}

And we can read it back out.
{{< highlight bash >}}
$ vault read secret/mysecrets
Key             	Value
---             	-----
refresh_interval	768h0m0s
name            	foo
password        	bar
{{< /highlight >}}

The `refresh_interval` is the least obvious field here, it is a hint to the client when they should check again to see if this data has changed. More information can be found at the page linked at the bottom of this section.

When I first tried this, I couldn't work out what I was seeing. Is this stuff encrypted or not, because this is way too easy! So some things to note:

* All of the data in the backend is encrypted at rest.
* All of the data is encrypted in transit.
* This dev server has given me root access to do anything I want (nearly).

You can also request the data in JSON format, which also gives a little more information.
{{< highlight bash >}}
$ vault read -format=json secret/mysecrets
{
	"request_id": "62999ca5-dbe7-f4d7-09dd-d75131ad3b2f",
	"lease_id": "",
	"lease_duration": 2764800,
	"renewable": false,
	"data": {
		"name": "foo",
		"password": "bar"
	},
	"warnings": null
}
{{< /highlight >}}

The secret can be deleted with a simple command.
{{< highlight bash >}}
$ vault delete secret/mysecrets
Success! Deleted 'secret/mysecrets' if it existed.
{{< /highlight >}}

**Note**: The kv backend **never** removes data on its own. Lease durations, TTLs and so on are all advisory for this backend, but we will talk about them more in the next sections.

You can read more about the kv backend here https://www.vaultproject.io/docs/secrets/kv/index.html.

##Encryption as a Service
The transit backend provides encryption of data, but does not store the result in the Vault, instead it returns the data to the caller. This can be useful if you want to keep the encryption keys in the vault but want to store the result in your applications primary data store.

First, mount the transit backend into Vault.
{{< highlight bash >}}
$ vault mount transit
Successfully mounted 'transit' at 'transit'!
{{< /highlight >}}

Then we need to create a cryptographically complex key to be used to sign all of our data. You can create multiple keys to be used for different things, so that if one is compromised the rest of your data is safe. Of course, obtaining the key is extremely unlikely since it is safely stored in Vault and never revealed.

{{< highlight bash >}}
$ vault write -f transit/keys/my-key
Success! Data written to: transit/keys/my-key

$ vault read transit/keys/my-key
Key                   	Value
---                   	-----
deletion_allowed      	false
derived               	false
exportable            	false
keys                  	map[1:1508678900]
latest_version        	1
min_decryption_version	1
min_encryption_version	0
name                  	my-key
supports_decryption   	true
supports_derivation   	true
supports_encryption   	true
supports_signing      	false
type                  	aes256-gcm96
{{< /highlight >}}

Now we can encrypt some data. The data you want to encrypt may not be plain text, so this backend requires that your data be base64.

{{< highlight bash >}}
$ vault write transit/encrypt/foo plaintext=$(echo 'perhaps_some_password' | base64)
Key       	Value
---       	-----
ciphertext	vault:v1:FmwcD8Bo4BWppinKOuDZ3t3oN/En/o7yXbpYGBgumEYdvKeDmLzZ6UYFunq+0/OAin8=
{{< /highlight >}}

Decrypting the value is as simple as calling transit/decrypt/...

{{< highlight bash >}}
$ vault write transit/decrypt/foo ciphertext=vault:v1:FmwcD8Bo4BWppinKOuDZ3t3oN/En/o7yXbpYGBgumEYdvKeDmLzZ6UYFunq+0/OAin8=
Key      	Value
---      	-----
plaintext	cGVyaGFwc19zb21lX3Bhc3N3b3JkCg==
{{< /highlight >}}

Remember, the value was base64 encoded. Lets decode it to get back to the original value.

{{< highlight bash >}}
$ echo "cGVyaGFwc19zb21lX3Bhc3N3b3JkCg==" | base64 --decode
perhaps_some_password
{{< /highlight >}}

##Dynamic Secret Leasing
This is the capability that really makes Vault stand apart from its rivals.

Vault is able to create credentials for various backends, on-request, and then revoke them again after some time. The upside is that you never need to create a static set of credentials for services like databases, cloud providers etc... that will be left around unless manually rotated. The credentials can exist for the amount of time they're are needed and then destroyed, minimizing the amount of damage any compromised credentials may cause.

In this section I will show setting up MySQL and AWS credentials.

##MySQL
First, mount the database backend.
{{< highlight bash >}}
$ vault mount database
Successfully mounted 'database' at 'database'!
{{< /highlight >}}

Next, lets configure the database backend. The connection URL is in the Data Source Name format used in the Go database packages https://github.com/go-sql-driver/mysql#dsn-data-source-name.  Here we are setting up a read only connection, perhaps pointing to a read slave server.
{{< highlight bash >}}
$ vault write database/config/mysql \
    plugin_name=mysql-database-plugin \
    connection_url="root:root@tcp(127.0.0.1:3306)/" \
    allowed_roles="readonly"

The following warnings were returned from the Vault server:
* Read access to this endpoint should be controlled via ACLs as it will return the connection details as is, including passwords, if any.
{{< /highlight >}}

That message is very important, the root MySQL configuration is not protected by default. You must configure ACLs to keep it safe. This topic is not covered in this article.

Now we need to tell Vault how to create a user in MySQL when an application requests readonly database access. The tokens inside the {{}} brackets will be dynamically filled in by Vault when creating each instance of a user/credentials. Note that we are setting the TTL settings for this connection, more information can be found on this in the excellent Vault documentation.
{{< highlight bash >}}
$ vault write database/roles/readonly \
    db_name=mysql \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"

Success! Data written to: database/roles/readonly
{{< /highlight >}}

Now an application can request a lease for some dynamic readonly credentials for MySQL. The username and password seen here is now a valid username and password for the specified MySQL database. Vault will delete the user after 1 hour.

**Note**: Where possible, Vault will rely on the backend's own system for automatically purging credentials, in the event that Vault is DDOS'd and unable to revoke itself.
{{< highlight bash >}}
$ vault read database/creds/readonly
Key            	Value
---            	-----
lease_id       	database/creds/readonly/59e6797f-c064-aaa0-10d4-43533ee001fe
lease_duration 	1h0m0s
lease_renewable	true
password       	A1a-5uxrszsxvr37uupv
username       	v-root-readonly-s60rvq7vvutt0z33

$ mysql -uroot -proot -e "SELECT * FROM mysql.user"
+-----------+----------------------------------+-------------+-------------+-------------+-------------+-------------+-----------+-------------+---------------+--------------+-----------+------------+-----------------+------------+------------+--------------+------------+-----------------------+------------------+--------------+-----------------+------------------+------------------+----------------+---------------------+--------------------+------------------+------------+--------------+------------------------+----------+------------+-------------+--------------+---------------+-------------+-----------------+----------------------+-----------------------+-------------------------------------------+------------------+-----------------------+-------------------+----------------+
| Host      | User                             | Select_priv | Insert_priv | Update_priv | Delete_priv | Create_priv | Drop_priv | Reload_priv | Shutdown_priv | Process_priv | File_priv | Grant_priv | References_priv | Index_priv | Alter_priv | Show_db_priv | Super_priv | Create_tmp_table_priv | Lock_tables_priv | Execute_priv | Repl_slave_priv | Repl_client_priv | Create_view_priv | Show_view_priv | Create_routine_priv | Alter_routine_priv | Create_user_priv | Event_priv | Trigger_priv | Create_tablespace_priv | ssl_type | ssl_cipher | x509_issuer | x509_subject | max_questions | max_updates | max_connections | max_user_connections | plugin                | authentication_string                     | password_expired | password_last_changed | password_lifetime | account_locked |
+-----------+----------------------------------+-------------+-------------+-------------+-------------+-------------+-----------+-------------+---------------+--------------+-----------+------------+-----------------+------------+------------+--------------+------------+-----------------------+------------------+--------------+-----------------+------------------+------------------+----------------+---------------------+--------------------+------------------+------------+--------------+------------------------+----------+------------+-------------+--------------+---------------+-------------+-----------------+----------------------+-----------------------+-------------------------------------------+------------------+-----------------------+-------------------+----------------+
| localhost | root                             | Y           | Y           | Y           | Y           | Y           | Y         | Y           | Y             | Y            | Y         | Y          | Y               | Y          | Y          | Y            | Y          | Y                     | Y                | Y            | Y               | Y                | Y                | Y              | Y                   | Y                  | Y                | Y          | Y            | Y                      |          |            |             |              |             0 |           0 |               0 |                    0 | mysql_native_password |                                           | N                | 2017-10-19 13:39:57   |              NULL | N              |
| localhost | mysql.session                    | N           | N           | N           | N           | N           | N         | N           | N             | N            | N         | N          | N               | N          | N          | N            | Y          | N                     | N                | N            | N               | N                | N                | N              | N                   | N                  | N                | N          | N            | N                      |          |            |             |              |             0 |           0 |               0 |                    0 | mysql_native_password | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | N                | 2017-10-19 13:39:57   |              NULL | Y              |
| localhost | mysql.sys                        | N           | N           | N           | N           | N           | N         | N           | N             | N            | N         | N          | N               | N          | N          | N            | N          | N                     | N                | N            | N               | N                | N                | N              | N                   | N                  | N                | N          | N            | N                      |          |            |             |              |             0 |           0 |               0 |                    0 | mysql_native_password | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | N                | 2017-10-19 13:39:57   |              NULL | Y              |
| %         | v-root-readonly-s60rvq7vvutt0z33 | Y           | N           | N           | N           | N           | N         | N           | N             | N            | N         | N          | N               | N          | N          | N            | N          | N                     | N                | N            | N               | N                | N                | N              | N                   | N                  | N                | N          | N            | N                      |          |            |             |              |             0 |           0 |               0 |                    0 | mysql_native_password | *5657F7F3731E1FBD45AB040AF5F18919C2717B63 | N                | 2017-10-23 08:15:08   |              NULL | N              |
+-----------+----------------------------------+-------------+-------------+-------------+-------------+-------------+-----------+-------------+---------------+--------------+-----------+------------+-----------------+------------+------------+--------------+------------+-----------------------+------------------+--------------+-----------------+------------------+------------------+----------------+---------------------+--------------------+------------------+------------+--------------+------------------------+----------+------------+-------------+--------------+---------------+-------------+-----------------+----------------------+-----------------------+-------------------------------------------+------------------+-----------------------+-------------------+----------------+
{{< /highlight >}}

After one hour, the credentials will be removed from the database. However, you can revoke the details with a command and the lease_id if required.

{{< highlight bash >}}
$ vault revoke database/creds/readonly/59e6797f-c064-aaa0-10d4-43533ee001fe
Success! Revoked the secret with ID 'database/creds/readonly/59e6797f-c064-aaa0-10d4-43533ee001fe', if it existed.

$ mysql -uroot -proot -e "SELECT * FROM mysql.user"
+-----------+---------------+-------------+-------------+-------------+-------------+-------------+-----------+-------------+---------------+--------------+-----------+------------+-----------------+------------+------------+--------------+------------+-----------------------+------------------+--------------+-----------------+------------------+------------------+----------------+---------------------+--------------------+------------------+------------+--------------+------------------------+----------+------------+-------------+--------------+---------------+-------------+-----------------+----------------------+-----------------------+-------------------------------------------+------------------+-----------------------+-------------------+----------------+
| Host      | User          | Select_priv | Insert_priv | Update_priv | Delete_priv | Create_priv | Drop_priv | Reload_priv | Shutdown_priv | Process_priv | File_priv | Grant_priv | References_priv | Index_priv | Alter_priv | Show_db_priv | Super_priv | Create_tmp_table_priv | Lock_tables_priv | Execute_priv | Repl_slave_priv | Repl_client_priv | Create_view_priv | Show_view_priv | Create_routine_priv | Alter_routine_priv | Create_user_priv | Event_priv | Trigger_priv | Create_tablespace_priv | ssl_type | ssl_cipher | x509_issuer | x509_subject | max_questions | max_updates | max_connections | max_user_connections | plugin                | authentication_string                     | password_expired | password_last_changed | password_lifetime | account_locked |
+-----------+---------------+-------------+-------------+-------------+-------------+-------------+-----------+-------------+---------------+--------------+-----------+------------+-----------------+------------+------------+--------------+------------+-----------------------+------------------+--------------+-----------------+------------------+------------------+----------------+---------------------+--------------------+------------------+------------+--------------+------------------------+----------+------------+-------------+--------------+---------------+-------------+-----------------+----------------------+-----------------------+-------------------------------------------+------------------+-----------------------+-------------------+----------------+
| localhost | root          | Y           | Y           | Y           | Y           | Y           | Y         | Y           | Y             | Y            | Y         | Y          | Y               | Y          | Y          | Y            | Y          | Y                     | Y                | Y            | Y               | Y                | Y                | Y              | Y                   | Y                  | Y                | Y          | Y            | Y                      |          |            |             |              |             0 |           0 |               0 |                    0 | mysql_native_password |                                           | N                | 2017-10-19 13:39:57   |              NULL | N              |
| localhost | mysql.session | N           | N           | N           | N           | N           | N         | N           | N             | N            | N         | N          | N               | N          | N          | N            | Y          | N                     | N                | N            | N               | N                | N                | N              | N                   | N                  | N                | N          | N            | N                      |          |            |             |              |             0 |           0 |               0 |                    0 | mysql_native_password | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | N                | 2017-10-19 13:39:57   |              NULL | Y              |
| localhost | mysql.sys     | N           | N           | N           | N           | N           | N         | N           | N             | N            | N         | N          | N               | N          | N          | N            | N          | N                     | N                | N            | N               | N                | N                | N              | N                   | N                  | N                | N          | N            | N                      |          |            |             |              |             0 |           0 |               0 |                    0 | mysql_native_password | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | N                | 2017-10-19 13:39:57   |              NULL | Y              |
+-----------+---------------+-------------+-------------+-------------+-------------+-------------+-----------+-------------+---------------+--------------+-----------+------------+-----------------+------------+------------+--------------+------------+-----------------------+------------------+--------------+-----------------+------------------+------------------+----------------+---------------------+--------------------+------------------+------------+--------------+------------------------+----------+------------+-------------+--------------+---------------+-------------+-----------------+----------------------+-----------------------+-------------------------------------------+------------------+-----------------------+-------------------+----------------+
{{< /highlight >}}


##AWS
Vault also has a backend for generating AWS credentials on the fly.

Like before, start by mounting the backend.
{{< highlight bash >}}
$ vault mount aws
Successfully mounted 'aws' at 'aws'!
{{< /highlight >}}

Now we configure the backend with the credentials it needs to create more users and roles.
{{< highlight bash >}}
$ vault write aws/config/root \
    access_key={AWS_ACCESS_KEY} \
    secret_key={AWS_SECRET_KEY}
Success! Data written to: aws/config/root
{{< /highlight >}}

The AWS backend differs from the database backend by restricting all access to the AWS config, even if you are root.
{{< highlight bash >}}
$ vault read aws/config/root
Error reading aws/config/root: Error making API request.

URL: GET http://127.0.0.1:8200/v1/aws/config/root
Code: 405. Errors:

* unsupported operation
{{< /highlight >}}

One more step needed for AWS, is to grant Vault a policy it can use to create IAM users. Be sure not to grant it any more capabilities than it needs. Save this to a file called `policy.json`.
{{< highlight json >}}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1426528957000",
      "Effect": "Allow",
      "Action": [
        "ec2:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
{{< /highlight >}}

Now we create a role for AWS with the given policy. The @ sign here tells the command line to load the data from a file.
{{< highlight bash >}}
$ vault write aws/roles/deploy policy=@policy.json
Success! Data written to: aws/roles/deploy
{{< /highlight >}}

Like with the MySQL backend, we can now read from and revoke credentials. As we do so, IAM users are created and destroyed in the AWS account.
{{< highlight bash >}}
$ vault read aws/creds/deploy
Key             Value
---             -----
lease_id        aws/creds/deploy/0d042c53-aa8a-7ce7-9dfd-310351c465e5
lease_duration  768h0m0s
lease_renewable true
access_key      AKIAJFN42DVCQWDHQYHQ
secret_key      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
security_token  <nil>

$ vault revoke aws/creds/deploy/0d042c53-aa8a-7ce7-9dfd-310351c465e5
Success! Revoked the secret with ID 'aws/creds/deploy/0d042c53-aa8a-7ce7-9dfd-310351c465e5', if it existed.
{{< /highlight >}}

#Closing Thoughts
Hopefully, I've demonstrated here that Vault could completely change the way we think about security. No more manual rotation of keys, no static database credentials, primarily automated access to AWS can all massively reduce the risk of a serious data breach or an AWS hijacking by Bitcoin bandits.

Have you used Vault in production? If so, please comment and tell me about your experience!
