---
title: Thales CipherTrust Manager (formerly Gemalto KeySecure)
date: 2023-02-08
lastmod: :git
draft: false
tableOfContents: true
---

This tutorial shows how to setup a KES server that uses a [Thales CipherTrust Manager](https://cpl.thalesgroup.com/encryption/ciphertrust-manager) instance (formerly known as Gemalto KeySecure) as a persistent and secure key store.

```goat
                 +---------------------------------------------+
 .----------.    |    .----------.     .-------------------.   |
| KES Client +---+---+ KES Server +---+ CipherTrust Manager │  |
 '----------'    |    '----------'     '-------------------'   |
                 +---------------------------------------------+
```

This guide assumes that you have a running CipherTrust Manager instance. 
It has been tested with CipherTrust Manager `k170v` version `2.0.0` and Gemalto KeySecure `k170v` version `1.9.1` and `1.10.0`. 

## CipherTrust Manager Setup

To connect to your CipherTrust Manager instance via the `ksctl` CLI you need a `config.yaml` file similar to:

```yaml
KSCTL_URL: <your-keysecure-endpoint>
KSCTL_USERNAME: <your-user/admin-name>
KSCTL_PASSWORD: <your-user/admin-password>
KSCTL_VERBOSITY: false
KSCTL_RESP: json
KSCTL_NOSSLVERIFY: true
KSCTL_TIMEOUT: 30
```

{{< admonition type="note">}}
Please make sure to use correct values for `KSCTL_URL`, `KSCTL_USERNAME` and `KSCTL_PASSWORD`
If your CipherTrust Manager instance has been configured with a TLS certificate trusted by your machine, then you can also set `KSCTL_NOSSLVERIFY: false`.
{{< /admonition >}}

1. Create a new group for KES
   
   ```sh
   ksctl groups create --name KES-Service
   ```

2. Create a new user for the group

   {{< admonition type="tip">}}
   This prints a JSON object containing a `user_id` needed for a later step.
   If you already have an existing user that you want to assign to the `KES-Service` group, skip this step and proceed with 3.
   {{< /admonition>}}

   ```sh
   ksctl users create --name <username> --pword '<password>'
   ```

3. Assign the user to the `KES-Service` group created in step 1
 
   ```sh
   ksctl groups adduser --name KES-Service --userid "<user-ID>"
   ```

   The user ID prints when creating the user.
   Otherwise, obtain the ID with the `ksctl users list` command. 

   A user-ID is similar to: `local|8791ce13-2766-4948-a828-71bac67131c9`.

5. Create a policy for the `KES-Service` group
  
   Create a text file named `kes-policy.json` that grants members of the `KES-Service` group **create**, **read** and **delete** permissions.
   THe contents of the file should be similar to the following:
   
   ```json
   {                                                                                          
     "allow": true,
     "name": "kes-policy",
     "actions":[
         "CreateKey",
         "ExportKey",
         "ReadKey",
         "DeleteKey"
     ],
     "resources": [
         "kylo:kylo:vault:secrets:*"
     ]
   }
   ```

   {{< admonition type="note">}}
   This policy allows KES to create, fetch and delete master keys. 
   If you want to prevent KES from e.g. deleting master keys omit the **DeleteKey** action.
   
   Similarly, you can restrict the master keys that can be accessed by KES via the `resources` definition.
   {{< /admonition >}}

   Use the following command to create the policy using the file created above.

   ```sh
   ksctl policy create --jsonfile kes-policy.json
   ```

6. Attach the policy to the `KES-Service` group
   
   Create a file named `kes-attachment.json` with the policy attachment specification:

   ```json
   {                                                                                          
      "cust": {
         "groups": ["KES-Service"]
      }
   }
   ```

   Use the following command to attach the `kes-policy` to the `KES-Service` group:

   ```sh
   ksctl polattach create -p kes-policy -g kes-attachment.json
   ```

8. Create a refresh token for the KES server to use to obtain short-lived authentication tokens.
   
   The following command returns a new refresh token:

   ```sh
   ksctl tokens create --user <username> --password '<password>' --issue-rt | jq -r .refresh_token
   ```
   
   Replace `<username>` and `<password>` with the credentials for a user that is a member of the `KES-Service` group.

   The command outputs a refresh token similar to

   ```
   CEvk5cdHLG7si05LReIeDbXE3PKD082YdUFAnxX75md3jzV0BnyHyAmPPJiA0
   ```

## KES Server Setup

The KES Server requires a TLS private key and certificate.

The KES server is [secure-by-default](https://en.wikipedia.org/wiki/Secure_by_default) and can only run with TLS. 
This tutorial uses self-signed certificates for simplicity.

{{< admonition type="note" >}}
For a production setup we highly recommend to use a certificate signed by trusted Certificate Authority.
This can be either your internal CA or a public CA such as [Let's Encrypt](https://letsencrypt.org).
{{< /admonition >}}

1. Generate a TLS private key and certificate for the KES server
 
   The following command generates a new TLS private key `server.key` and a self-signed X.509 certificate `server.cert` that is issued for the IP `127.0.0.1` and DNS name `localhost` (as SAN). 
   Customize the command to match your setup.
   
   ```sh
    kes tool identity new --server --key server.key --cert server.cert --ip "127.0.0.1" --dns localhost
   ```
   
   {{< admonition type="tip" >}}
   Any other tooling for X.509 certificate generation works as well. 
   For example, you could use `openssl`:
   
   ```sh
   $ openssl ecparam -genkey -name prime256v1 | openssl ec -out server.key
   
   $ openssl req -new -x509 -days 30 -key server.key -out server.cert \
       -subj "/C=/ST=/L=/O=/CN=localhost" -addext "subjectAltName = IP:127.0.0.1"
   ```
   {{< /admonition >}}

2. Create a private key and certificate for the application
 
   ```sh
   kes tool identity new --key=app.key --cert=app.cert app
   ```

   You can compute the `app` identity at any time.

   ```sh
   kes tool identity of app.cert
   ```

3. Create the [config file]({{< relref "/tutorials/configuration.md#config-file" >}}) `server-config.yml`

   ```yaml
   address: 0.0.0.0:7373
   root:    disabled  # We disable the root identity since we don't need it in this guide 

   tls:
     key:  server.key
     cert: server.cert

   policy:    
     my-app: 
       allow:
       - /v1/key/create/my-app*
       - /v1/key/generate/my-app*
       - /v1/key/decrypt/my-app*
       identities:
       - ${APP_IDENTITY}

   keystore:
     gemalto:
       keysecure:
         endpoint: ""  # The REST API endpoint of your KeySecure instance - e.g. https://127.0.0.1
         credentials:
           token:  ""  # Your refresh token
           domain: ""  # Your domain. If empty, defaults to root domain.
           retry:  15s
         tls:
           ca: "" # Optionally, specify the certificate of the CA that issued the KeySecure TLS certificate.
   ```

   Use your refreshed token.

4. Start a KES server in a new window/tab:  

   ```sh
   export APP_IDENTITY=$(kes tool identity of app.cert)
   
   kes server --config=server-config.yml --auth=off
   ```
   
   {{< admonition type="note">}}
   The command uses `--auth=off` because our `root.cert` and `app.cert` certificates are self-signed.
   {{< /admonition >}}

   If starting the server fails with an error message similar to:

   ```sh
   x509: certificate is not valid for any names, but wanted to match <your-endpoint>
   ```

   then your CipherTrust Manager instance serves a TLS certificate with neither a common name (subject) nor a subject alternative name (SAN). 
   Such a certificate is invalid. 
   Update the TLS certificate of your CipherTrust Manager instance. 
   
   You can analyze a certificate with: `openssl x509 -text -noout <certificate>`

5. In the other window or tab, connect to the server
 
   ```sh
   export KES_CLIENT_CERT=app.cert
   export KES_CLIENT_KEY=app.key
   kes key create -k my-app-key
   ```

   ```sh
   export APP_IDENTITY=$(kes tool identity of app.cert)
   
   kes server --config=server-config.yml --auth=off
   ```
   
   {{< admonition type="note">}}
   The command uses `--auth=off` because our `root.cert` and `app.cert` certificates are self-signed.
   {{< /admonition >}}

6. Derive and decrypt data keys from the previously created `my-app-key`
 
   ```sh
   kes key derive -k my-app-key
   {
     plaintext : ...
     ciphertext: ...
   }
   ```
   ```sh
   kes key decrypt -k my-app-key <base64-ciphertext>
   ```