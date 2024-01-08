# Securing the Docker daemon with TLS

**Instructions for securing the Docker daemon with a self-signed certificate**

*The following instructions are designed to be executed on the host machine of the Docker daemon. With some modification, these insturctions were sourced from the official [Docker Documentation: Protecting the Docker daemon socket.](https://docs.docker.com/engine/security/protect-access/)*

*StackOverflow was also instrumental in helping me to resolve some of the issues absent in the official Docker docs.
[StackOverflow: Unable to start docker after configuring hosts in daemon.json](https://stackoverflow.com/questions/44052054/unable-to-start-docker-after-configuring-hosts-in-daemon-json)*

## Create CA server and client keys with OpenSSL

### First, let's initialize some variables that will make this process go a bit more smoothly.

```bash
# Change this to the IP address of your docker host
export DOCKER_HOST_IP="0.0.0.0"

# Set the hostname of your docker host.
# Include the subdomain if your docker host has one.
# `DOCKER_HOST_FQDN="my-subdomain.$(hostname -f)"`
# if `hostname -f` doesn't include it.
# Or, just type it in explicitly.
# `DOCKER_HOST_FQDN="my-subdomain.my-hostname.my-tld"`
export DOCKER_HOST_FQDN="$(hostname -f)"

# We'll need this to set up TLS for `docker.service`.
export DOCKER_SERVICE_DIR="/etc/systemd/system/docker.service.d"

# A directory we will create to store the
# Certificate Authority (CA) server credentials.
export DOCKER_TLS_DIR="/etc/docker/.tls"

# The `subjectAltName` for your CA credentials
export CERT_SAN_CONFIGURATOIN="$DOCKER_HOST_FQDN,IP:$DOCKER_HOST_IP,IP:127.0.0.1"

# Set the docker host
export DOCKER_HOST=tcp://$DOCKER_HOST_FQDN:2375

# And finally, set `tlsverify` to `true`
# Set this to `0` if for any reason you need to disable `tlsverify`.
# For instance, if find yourself in a position at the end of this
# process in which you need access the Docker daemon (docker.service)
# without TLS.
export DOCKER_TLS_VERIFY=1
```

### Now, let's create the key and certificate signing request (CSR) for the server.

**1. Create and enter the working directory for the creation of your credentials.**

```bash
mkdir -v ~/.docker/tls/ && cd ~/.docker/tls/
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
openssl req -subj "/CN=$DOCKER_HOST_FQDN" -sha256 -new -key server-key.pem -out server.csr
```

**5. Sign the public key with our Certificate Authority.**

Create the configuration file that will inform `opnessl` of the Subject Alt Name using the FQDN your certificate is for.

For the top level domain set the FQDN and the IPs of the Docker host machine. *Replace the first IP address with the IP address of your host machine.*

```bash
echo -e "subjectAltName=DNS:$CERT_SAN_CONFIGURATOIN" > server.cnf
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

*Again, the following instructions are designed to be executed on the host machine of the Docker daemon. We continue the following instructions in the same working directory as before,* `~/.docker/tls`*.*

**1. Create the client key and certificate signing request.**

First, the key...

```bash
openssl genrsa -out key.pem 4096
```

Then, create the client's certificate signing request.

```bash
openssl req -subj '/CN=client' -new -key key.pem -out client.csr
```

**2. Set the key for client authentication.**

```bash
echo extendedKeyUsage = clientAuth > client-auth.cnf
```

**3. Generate the client's signed certificate.**

```bash
openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
-CAcreateserial -out cert.pem -extfile client-auth.cnf
```

And enter your pass phrase.

```bash
Signature ok
subject=/CN=client
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```

**4. Gabage colletion.**

Once we've generated the `cert.pem` and `server-cert.pem` credentials, we can safely remove the certificate signing requests and `.cnf` files.

```bash
rm -v client.csr server.csr server.cnf client-auth.cnf
```

**5. Setting permissions.**
If the `umask` of the machine you're using to create the credentials is set to 022, your secret keys are world-readable and writable for you and your group. You can learn more about setting your `umask` [here](https://www.linuxnix.com/umask-define-linuxunix/). Let's protect our credentials from accidental corruption by setting the proper permissions.

For your eyes only...

```bash
chmod -v 0400 ca-key.pem key.pem server-key.pem
```

For the world to see, but not to change...

```bash
chmod -v 0444 ca.pem server-cert.pem cert.pem
```

Finally, we have all the necessary credentials for the server and clients that need to connect to the server. Next, we need to configure the Docker daemon, `dockerd`, to only accept connections from clients that provide a trusted certificate, issued by your Certificate Authority. (CA).

---

## Secure Docker with TLS verification

**1. Configure `dockerd` for TLS using the server credentials you created earlier.**

NOTE: I found the following to be necessary for Debian and Ubuntu distros of Linux. You may not be able to access your Docker daemon over TLS without this modifcation.

Here, we are not changing any existing files in the systemd service directories. We are simply adding an override configuration that will you to enable TLS for the Docker daemon.

First, create the `override.conf` file for the `docker.service`.

```bash
cat > ~/.docker/override.conf <<EOL
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375
EOL
```

Next, create the service directory to allow the Docker daemon use of your new configuration.

```bash
sudo mkdir -v /etc/systemd/system/docker.service.d
```

If you get the following warning then the directory already exists. So, all we need to do next is copy the new configuration file to that directory, reload the systemd daemon, and restart the Docker daemon.

```warning
mkdir: cannot create directory ‘/etc/systemd/system/docker.service.d’: File exists
```

Now, copy the `override.conf` file to the docker service directory and reload the system daemon.

```bash
sudo cp -v override.conf /etc/systemd/system/docker.service.d/
sudo systemctl daemon-reload
```

Later, we will restart the `docker.service`. Before we do that, we need to make the server credentials available to the Docker daemon. 

**2. Create a directory for the credentials for the server and copy the credentials into the new directory.**

```bash
cd ~/.docker/tls
sudo mkdir -v /etc/docker/.tls
sudo cp -v {ca,server-cert,server-key}.pem /etc/docker/.tls/
```

Now that the credentials are in place, we need to create the `daemon.json` for the `docker.service`.

**NOTE:** *If the* `/etc/docker/daemon.json` *already exists, then do not run the following* `cat` *command. Simply open your existing* `/etc/docker/daemon.json` *in your preferred editor and add the following* `json` *data to your existing* `/etc/docker/daemon.json`*.* 

```bash
cat > ~/.docker/daemon.json <<EOL
{
  "tlsverify": true,
  "tlscacert": "/etc/docker/.tls/ca.pem",
  "tlscert": "/etc/docker/.tls/server-cert.pem",
  "tlskey": "/etc/docker/.tls/server-key.pem"
}
EOL
```

Now, we can reload the system daemon and restart `docker.service`.

```bash
sudo systemctl restart docker
```

Next, copy your client credentials to your `~/.docker/` directory.

```bash
cp -v {ca,cert,key}.pem ~/.docker
```

*Replace `[HOST]` with the FQDN of your docker server as set in the server credentials you created earlier.*

```bash
export DOCKER_HOST_FQDN=[HOST]

docker --tlsverify \
--tlscacert=ca.pem \
--tlscert=client-cert.pem \
--tlskey=client-key.pem \
-H=$DOCKER_HOST_FQDN:2375 version
```

**3. Secure the connection by default.**

```bash
cp -v {ca,client-cert,client-key}.pem ~/.docker
export DOCKER_HOST=tcp://$DOCKER_HOST_FQDN:2375 DOCKER_TLS_VERIFY=1
```

---

### Other Connection Modalities

You can read more about other options for connecting to a secure Docker daemon in the official  [Docker Documentation](https://docs.docker.com/engine/security/protect-access/#other-modes).

---
