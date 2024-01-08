# Securing the Docker Daemon with TLS

**Instructions for securing the Docker daemon with a self-signed certificate**

*The following instructions are designed to be executed on the host machine of the docker daemon. With some modification, these insturctions were sourced from the official [Docker Documentation](https://docs.docker.com/engine/security/protect-access/): Protecting the Docker daemon socket.*

## Create CA server and client keys with OpenSSL

### First, let's create the key and certificate signing request (CSR) for the server.

**1. Create and enter the working directory for the creation of your credentials.**

```bash
mkdir ~/.docker/tls/ && \
cd ~/.docker/tls/
```

**2. Generate your private and public keys**

```bash
openssl genrsa -aes256 -out ca-key.pem 4096
```

Then, enter a passphrase that will be used to encrypt the key, and enter it again to verify your input.

```bash
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
```

Next, create your public key.

```bash
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
```

Enter the pass phrase you used to generate your private key, and follow the prompts.

```bash
Enter pass phrase for ca-key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]: US
State or Province Name (full name) [Some-State]: Virginia
Locality Name (eg, city) []: Alexandria
Organization Name (eg, company) [Internet Widgits Pty Ltd]: ACME, Inc.
Organizational Unit Name (eg, section) []: Sales
Common Name (e.g. server FQDN or YOUR name) []: www.acmeinc.com
Email Address []: webmaster@acmeinc.com
```

**4. Create a server key and certificate signing request (CSR) for the docker host machine.**

Frist, the key...

```bash
openssl genrsa -out server-key.pem 4096
```

Now, the certificate signing request...

```bash
openssl req -subj "/CN=$(hostname -f)" -sha256 -new -key server-key.pem -out server.csr
```

Or, if you need to specify a subdomain...

```bash
openssl req -subj "/CN=my-subdomain.$(hostname -f)" -sha256 -new -key server-key.pem -out server.csr
```

**5. Sign the public key with our Certificate Authority.**

Create the configuration file that will inform `opnessl` of the Subject Alt Name using the FQDN your certificate is for.

For the top level domain set the FQDN and the IPs of the Docker host machine. *Replace the first IP address with the IP address of your host machine.*

```bash
echo -e "subjectAltName=DNS:$(hostname -f),IP:10.8.16.32,IP:120.0.0.1" > server.cnf
```

Or, if you need to use a subdomain...

```bash
echo -e "subjectAltName=DNS:my-hostname.$(hostname -f),IP:10.8.16.32,IP:120.0.0.1" > server.cnf
```

Next, set the Docker daemon key's extended usage attributes to be used only for server authentication.

```bash
echo extendedKeyUsage = serverAuth >> server.cnf
```

Finally, generate the signed certificate...

```bash
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
-CAcreateserial -out server-cert.pem -extfile server.cnf
```

...and enter the same pass phrase you used at the start.

```bash
Signature ok
subject=/CN=your.host.com
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```

That takes care of the key and certificate signing request for the server.

---

### Now, let's create the key and certificate signing request (CSR) for the client.

*Again, the following instructions are designed to be executed on the host machine of the Docker daemon. We continue the following instructions in the same working directory as before, `~/.docker/tls`*

**1. Create the client key and certificate signing request.**

First, the key...

```bash
openssl genrsa -out client-key.pem 4096
```

Then, create the client's certificate signing request.

```bash
openssl req -subj '/CN=client' -new -key client-key.pem -out client.csr
```

**2. Set the key for client authentication.**

```bash
echo extendedKeyUsage = clientAuth > client-auth.cnf
```

**3. Generate the client's signed certificate.**

```bash
openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
-CAcreateserial -out client-cert.pem -extfile client-auth.cnf
```

And enter your pass phrase.

```bash
Signature ok
subject=/CN=client
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```

**4. Gabage colletion.**

Once we've generated the `client-cert.pem` and `server-cert.pem` credentials, we can safely remove the certificate signing requests and `.cnf` files.

```bash
rm -v client.csr server.csr server.cnf client-auth.cnf
```

**5. Setting permissions.**
If the `umask` of the machine you're using to create the credentials is set to 022, your secret keys are world-readable and writable for you and your group. You can learn more about setting your `umask` [here](https://www.linuxnix.com/umask-define-linuxunix/). Let's protect our credentials from accidental corruption by setting the proper permissions.

For your eyes only...

```bash
chmod -v 0400 ca-key.pem client-key.pem server-key.pem
```

For the world to see, but not to change...

```bash
chmod -v 0444 ca.pem server-cert.pem client-cert.pem
```

Finally, we have all the necessary credentials for the server and clients that need to connect to the server. Next, we need to configure the Docker daemon, `dockerd`, to only accept connections from clients that provide a trusted certificate, issued by your Certificate Authority. (CA).

---

## Secure Docker with TLS verification 

**1. Configure `dockerd` for TLS using the server credentials you created earlier.**

```bash
dockerd \
--tlsverify \
--tlscacert=ca.pem \
--tlscert=server-cert.pem \
--tlskey=server-key.pem \
-H=0.0.0.0:2376
```

**2. Test the connection with the client credentials you just created.**

*Replace `[HOST]` with the FQDN of your docker server as set in the server credentials you created earlier.*

```bash
export DOCKER_HOST_FQDN=[HOST]

docker --tlsverify \
--tlscacert=ca.pem \
--tlscert=client-cert.pem \
--tlskey=client-key.pem \
-H=$DOCKER_HOST_FQDN:2376 version
```

**3. Secure the connection by default.**

```bash
cp -v {ca,client-cert,client-key}.pem ~/.docker
export DOCKER_HOST=tcp://$DOCKER_HOST_FQDN:2376 DOCKER_TLS_VERIFY=1
```

---

### Other Connection Modalities

You can read more about other options for connecting to a secure Docker daemon in the official  [Docker Documentation](https://docs.docker.com/engine/security/protect-access/#other-modes).
