---
title: Containerised Postgress with Cert Auth. 
author: octi
date: 2024-03-26 12:00:00 +0800
categories: [Projects, MockUpBank]
tags: [DevSecOps, Tutorial, Postgres]
comments: false
toc: true
---

I don't like passwords. Even less I like paintext passwords in console commands and configs. The other day I decided to add a sprinke of security to a dev psql server of the MockUpBank project and set myself down the rabbit hole. 
99.999% of the examples of containerised postgress have `- POSTGRES_PASSWORD=secret` either in a compose file, or as an argument in the command line. 
First I recalled that at one of my previos jobs we used [Mozilla SOPS](https://github.com/getsops) to encrypt the configs. Since it still requires baking secrets into the container image and only solves only one part of the problem - storing secrets at rest, but dosen't prevent MitM and provide robust athentication I decided to go for a full featured certificate based auth. 
To my surprise there appeared to be not so many tutorials that worked with my particular setup (MacOS, alpine based pg image). The caveat was in the access rights and ownerships of the .crt and .key files that I wanted to attach to the contianer as volumes. After several unsuccesfull attempts I decided to bake the image myself.

## Off we go
According to the official [postgres docs](https://www.postgresql.org/docs/current/ssl-tcp.html) we'll need these four files for the server:

| File                                   | Purpose                                                        |
|----------------------------------------|:---------------------------------------------------------------|
| ssl_cert_file ($PGDATA/server.crt)     | server certificate that indicates server's identity to the client |
| ssl_key_file ($PGDATA/server.key)      | server key for the server cert |
| ssl_ca_file                            | certificate of the Certification Authority that establishes the trust for both client and server certs |
| ssl_crl_file                           | certificate revokation list (speaks for itself) |

We are also going to need the client key and corresponding certificate in order to be able to connect to our server.

> I'm lazy, so I will describe a simplified set-up where the root CA cert is used to establish the trust rather than intermediary or leaf certs. In case of compromise such setup will require rebuilding the entire chain fo trust from grounds up - and it's pain. Don't do this in production!
{: .prompt-warning }

Instead of manipulating the certs and keys with openssl as prescribed by the official manual I decided to use a shortcut in the form of [certstrap](https://github.com/square/certstrap). We'll use it to generate keys and certs as well as sign and verify them. Tool installation steps are clear and not provided here  

> In prod environments one should consider [cfssl](https://github.com/cloudflare/cfssl) as a better alternaiteve to the certstrap, since it provides convinient certificate bundling options and an HTTP API server for signing, verifying, and bundling TLS certificates.
{: .prompt-tip }

## Establishing the Trust

Create the following directory structure for your own convinience.
```
└── db
    ├── conf                //Will hold container spec file and other configs 
    │   ├── pki
    │   │   ├── ca          // CA key and certs
    │   │   ├── client      // Client key and certs
    │   │   ├── out         // Default certstrap output dir
    │   │   └── server      // Server key and certs
```
In order to save some time copy the certstrap binary to the ./pki directory `cp certstrap db/conf/pki`. It is assumed that you will launch all the following commands from the pki dir. Let's initiate our CA with `./certstrap init --common-name "CertAuth"`. This will create three files in the pki/out directory:
- CertAuth.crt which is the CA certificate,
- CertAuth.key which is the CA certificate key that will sign certificate requests,
- CertAuth.crl which is the Certificate Revocation List(a list of revoked certificates). 

Now let's generate and sign the server key and cert. 

```bash
$ certstrap request-cert --common-name postgresdb  --domain localhost
$ certstrap sign postgresdb -CA CertAuth
```
> It is important to set the CN, or "common name" field, correctly. In verify-full mode which we will use the PostgreSQL client checks that the certificate the server presents is both signed by a trusted CA and that the CN on the server's certificate matches the host (-h) that the client requested to connect to. Since the container will be used localy we set up the "--domain localhost" option which adds a list of domains(called Subject Alternative Names) that the generated certificate will be valid for. 
{: .prompt-warning }

## Baking Custom Image

Although the most common practice for containerised postgres would be to attach a local volume with all the neccessary configuration files, I decided to also bake them into the image as I encountered the same issue with file ownership and ACLs as with certs.
Let's configre how PostgreSQL host-based authentication must work. Create a `pg_hba.conf` file inside db/conf and add the following lines there:
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                peer
local   all             pguser                                  peer
# do not let the "postgres" superuser login via a certificate
#hostssl all             postgres        ::/0                    reject
#hostssl all             postgres        0.0.0.0/0               reject
#
hostssl all             all             ::/0                    cert
hostssl all             all             0.0.0.0/0               cert
```
Peer based auth for superusers (postgress and pguser accounts in my case) is allowed only over a local connection and any remote connections for the postgres user is disallow. All other connections require a certificate.
Now let's create a new spec for our custome container. Create a `pg_container.spec` file inside db/conf and add the following lines there:
```
# This file contains the image specification of our database
# Base image
FROM postgres:16.2-alpine3.19

# Placing the server key and cert
COPY ./pki/server/postgresdb.key /var/lib/postgresql/server.key
COPY ./pki/server/postgresdb.crt /var/lib/postgresql/server.crt

# Placing the CA cert (not the key!!!) and the revokation list
COPY ./pki/ca/CertAuth.crt /var/lib/postgresql/ca.crt
COPY ./pki/ca/CertAuth.crl /var/lib/postgresql/ca.crl

# Placing custom HBA config
COPY ./pg_hba.conf /pgconf/pg_hba.conf

# Setting up the correct ownership and access modes for the keys and configs. The server will refuse to run if those are not set correctly.  
RUN chown 0:70 /var/lib/postgresql/server.key && chmod 640 /var/lib/postgresql/server.key
RUN chown 0:70 /var/lib/postgresql/server.crt && chmod 640 /var/lib/postgresql/server.crt

RUN chown 0:70 /var/lib/postgresql/ca.crt && chmod 640 /var/lib/postgresql/ca.crt
RUN chown 0:70 /var/lib/postgresql/ca.crl && chmod 640 /var/lib/postgresql/ca.crl

RUN chown 0:70 /pgconf/pg_hba.conf && chmod 640 /pgconf/pg_hba.conf

ENTRYPOINT ["docker-entrypoint.sh"] 
# Set up the 
CMD [ "-c", "ssl=on" , "-c", "ssl_cert_file=/var/lib/postgresql/server.crt", "-c",\
    "ssl_key_file=/var/lib/postgresql/server.key", "-c",\
    "ssl_ca_file=/var/lib/postgresql/ca.crt", "-c", "ssl_crl_file=/var/lib/postgresql/ca.crl", "-c",\
    "hba_file=/pgconf/pg_hba.conf" ]
```
After all the manipulations your final directory tree should look like this:
```bash
$ db tree
.
├── conf
│   ├── pg_container.spec
│   ├── pg_hba.conf
│   └── pki
│       ├── ca
│       │   ├── CertAuth.crl
│       │   ├── CertAuth.crt
│       │   └── CertAuth.key
│       ├── certstrap
│       ├── client
│       ├── out
│       │   ├── CertAuth.crl
│       │   ├── CertAuth.crt
│       │   ├── CertAuth.key
│       │   ├── postgresdb.crt
│       │   ├── postgresdb.csr
│       │   └── postgresdb.key
│       └── server
│           ├── postgresdb.crt
│           ├── postgresdb.csr
│           └── postgresdb.key
```
Now let's build our container. I use podman for this, the syntax is similar for docker: `podman build --rm -f "pg_container.spec" -t custom-pg16.2:ssl "."`

## Launch and Connect

Now that the image is ready let's run the container and test if our security gates are working as expected. Create a compose.yaml in the ./db directory and add the following:
```yaml
services:
  db:
    image: custom-pg16.2:ssl
    # set shared memory limit when using docker-compose
    shm_size: 128mb
    # or set shared memory limit when deploy via swarm stack
    #volumes:
    #  - type: tmpfs
    #    target: /dev/shm
    #    tmpfs:
    #      size: 134217728 # 128*2^20 bytes = 128Mb
    ports:
      - 5435:5432
    restart: on-failure:3
    environment:
      POSTGRES_PASSWORD: m@tters_no_m0r3
      POSTGRES_DB: postgres
      POSTGRES_USER: pguser
      PGSSLMODE: "verify-full"
```
Run `podman compose --file compose.yaml up`. The `[1] LOG:  database system is ready to accept connections` entry will signify your success. In case of errors casued by the file permissions or similar, resolve them, rebuild the image if needed. 
> You may run into a situation when the container is up and running but the database is still not accepting connections and requires changes to configuration files. For obvious reasons reloading the container won't help. The `$ podman exec -it {your_postgres_container_name}  psql -U postgres -c "SELECT pg_reload_conf();"` will come to your rescue and urge the postgres server to reload its configuration on the fly.
{: .prompt-tip } 
Try remote connection to the database with the user and password we've set up in the compoase.yaml. You should get a `FATAL:  password authentication failed for user "pguser"` error.

Let's move on to seting up our client. The CN for the client certificate must match the username of the client in the database (in our case it will be pguser). Generate and sign the key for the client:
```bash
$ certstrap request-cert --common-name pguser  --domain localhost
$ certstrap sign pguser --CA CertAuth
```
You can use either a console client or a GUI however the approach will be the same, you need to set up the following parameters:
- sslcert points to your client cert
- sslkey points to the corresponding client key
- sslrootcert points to the CA certificate
- sslmode is set to verify-full
- port is 5435
- database is the value of `POSTGRES_DB`
- username is the value of `POSTGRES_USER` 

This should be enogh for the server to let you in regardless of which password you provide for the pguser! Modifying the username, the host, sslrootcert or PGSSLMODE variables should lead to "Connection refused" errors. 
Here is an example for a `go migrate` command:
```bash
$ migrate -path db/migration -database "postgres://pguser:any@localhost:5435/simplebank?sslmode=verify-full&sslcert=db/conf/pki/out/pguser.crt&sslkey=db/conf/pki/out/pguser.key&sslrootcert=db/conf/pki/out/CertAuth.crt" -verbose up
```


