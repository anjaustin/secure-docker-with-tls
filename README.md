# Securing the Docker daemon with TLS

![Secure Docker with TLS](./assets/images/secure-docker-with-tls.png)

**Securing the Docker daemon socket with a self-signed certificate.**

*The following instructions should be executed on the host machine of the Docker daemon you wish to secure. With some modification, the following instructions were sourced from the official [Docker Documentation: Protecting the Docker daemon socket.](https://docs.docker.com/engine/security/protect-access/)*

*StackOverflow was also instrumental in helping me resolve some obscurities in the official Docker docs. [StackOverflow: Unable to start docker after configuring hosts in daemon.json](https://stackoverflow.com/questions/44052054/unable-to-start-docker-after-configuring-hosts-in-daemon-json)*

## Create CA server and client credentials with OpenSSL

### Let us initialize some variables.

```bash
# Change this to the IP address of your docker host
export DOCKER_HOST_IP="0.0.0.0"

# Set the hostname of your docker host.
# Include the subdomain if your docker host has one.
# Do `export DOCKER_HOST_FQDN="my-subdomain.$(hostname -f)"`,
# if `hostname -f` doesn't include it.
# Or, just type it in explicitly and export it.
# `export DOCKER_HOST_FQDN="subdomain.domain-name.com"`
export DOCKER_HOST_FQDN="$(hostname -f)"

# We'll need this to set up TLS for the `docker.service`.
export DOCKER_SERVICE_DIR="/etc/systemd/system/docker.service.d"

# A directory for our Certificate Authority (CA)
# server credentials.
export DOCKER_TLS_DIR="/etc/docker/.tls"

# The `subjectAltName` for your CA credentials. If the IP
# address of your host machine changes you'll need to recreate
# and reinstall your server credentials.
export CERT_SAN_CONFIGURATION="$DOCKER_HOST_FQDN,IP:$DOCKER_HOST_IP,IP:127.0.0.1"

# Set the docker host.
export DOCKER_HOST="tcp://$DOCKER_HOST_FQDN:2375"

# And finally, set `tlsverify` to `true`.
# Set this to `0` if you need to disable `tlsverify`.
# For instance, if find yourself in a position at the end of this
# process in which you need access to the Docker daemon (docker.service)
# without TLS, setting this to `0` will allow you to access the service.
# Otherwise, `1` defaults to TLS.
export DOCKER_TLS_VERIFY=1
```

### Now, let's create the server's key and certificate signing request (CSR).

**1. Create and enter the working directory for the creation of your credentials.**

```bash
mkdir -v ~/.docker/tls/ && cd ~/.docker/tls/
```

**2. Generate your private and public keys.**

Private key first.

```bash
openssl genrsa -aes256 -out ca-key.pem 4096
```

Enter a passphrase with which to encrypt the key, and enter it again to verify your passphrase.

```bash
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
```

Now, the public key.

```bash
openssl req -new -x509 -days 365 \
-key ca-key.pem -sha256 \
-out ca.pem
```

Enter the passphrase you used to generate your private key and follow the prompts.

```bash
Enter pass phrase for ca-key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields, but you can leave some blank
For some fields, there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2-letter code) [AU]: US
State or Province Name (full name) [Some-State]: Virginia
Locality Name (eg, city) []: Alexandria
Organization Name (eg, company) [Internet Widgits Pty Ltd]: ACME, Inc.
Organizational Unit Name (eg, section) []: Sales
Common Name (e.g. server FQDN or YOUR name) []: www.acmeinc.com
Email Address []: webmaster@acmeinc.com
```

**4. Create a server key and certificate signing request (CSR) for the docker host machine.**

First, the server key.

```bash
openssl genrsa -out server-key.pem 4096
```

And, the certificate signing request (CSR).

```bash
openssl req -subj "/CN=$DOCKER_HOST_FQDN" -sha256 -new \
-key server-key.pem \
-out server.csr
```

**5. Sign the public key with our Certificate Authority.**

Create the configuration file that will inform `opnessl` of the Subject Alt Name (SAN).

```bash
echo -e "subjectAltName=DNS:$CERT_SAN_CONFIGURATION" > server.cnf
```

Next, set the extended usage attribute of this key to server auth only.

```bash
echo extendedKeyUsage = serverAuth >> server.cnf
```

Generate the signed certificate.

```bash
openssl x509 -req -days 365 -sha256 \
-in server.csr \
-CA ca.pem \
-CAkey ca-key.pem \
-CAcreateserial -out server-cert.pem \
-extfile server.cnf
```

Finally, enter the same passphrase you used at the start.

```bash
Signature ok
subject=/CN=your.host.com
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```

That takes care of the key and certificate signing request for the server.

---

### Now, let's create the client's key and certificate signing request (CSR).

*Again, the following instructions should be executed on the host machine of the Docker daemon you wish to secure with TLS. We continue the following instructions in the same working directory as before,* `~/.docker/tls`*.*

**1. Create the client key and certificate signing request.**

First, the client key.

```bash
openssl genrsa -out key.pem 4096
```

Then, the client's certificate signing request.

```bash
openssl req -subj '/CN=client' -new \
-key key.pem \
-out client.csr
```

**2. Set the key for client authentication.**

```bash
echo extendedKeyUsage = clientAuth > client-auth.cnf
```

**3. Generate the client's signed certificate.**

```bash
openssl x509 -req -days 365 -sha256 \
-in client.csr \
-CA ca.pem \
-CAkey ca-key.pem \
-CAcreateserial -out cert.pem \
-extfile client-auth.cnf
```

Enter your passphrase.

```bash
Signature ok
subject=/CN=client
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```

**4. Garbage collection.**

Once we've generated the client's `cert.pem` and the `server-cert.pem` credentials, we can safely remove the certificate signing requests and `.cnf` files.

```bash
rm -v client.csr \
server.csr \
server.cnf \
client-auth.cnf
```

**5. Setting permissions.**

If the `umask` of the machine you're using is set to `022`, your secret keys are world-readable and writable for you and your group. You can learn more about setting your system's `umask` [here](https://www.linuxnix.com/umask-define-linuxunix/). Let's protect our credentials from accidental corruption by setting the proper permissions.

👀 ***For your eyes only.***

```bash
chmod -v 0400 ca-key.pem key.pem server-key.pem
```

For the world to see, not to change.

```bash
chmod -v 0444 ca.pem server-cert.pem cert.pem
```

Finally, we have all the required credentials for the server and for clients that need to connect to the server. Next, we need to configure the `docker.service` to accept connections from clients that provide a trusted certificate issued by your Certificate Authority (CA). You.

---

## Secure Docker with TLS verification

**1. Configure `docker.service` for TLS using the server credentials you created earlier.**

**NOTE:** *I found the following necessary for Debian and Ubuntu distros. You may not be able to access your* `docker.service` *over TLS without this modification.*

Here, we are not changing existing files in the systemd service directories. We are simply adding an override configuration to enable TLS for the `docker.service`.

First, create the `override.conf` file for the `docker.service`.

```bash
cat > ~/.docker/override.conf <<EOL
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375
EOL
```

Next, create the service directory for the `override.conf` file.

```bash
sudo mkdir -v /etc/systemd/system/docker.service.d
```

If you get the following warning, then the directory already exists. At which point, we copy the new configuration file to that directory, reload the systemd daemon, and restart the `docker.service`.

```warning
mkdir: cannot create directory ‘/etc/systemd/system/docker.service.d’: File exists
```

Copy the `override.conf` file to the docker service directory and reload the system daemon.

```bash
sudo cp -v override.conf /etc/systemd/system/docker.service.d/
sudo systemctl daemon-reload
```

Before we restart the `docker.service`, we need to make the server credentials available to the `docker.service`. 

**2. Create a directory for the server's credentials and copy the credentials into the new directory.**

```bash
cd ~/.docker/tls # ...just to be sure. :)
sudo mkdir -v /etc/docker/.tls
sudo cp -v {ca,server-cert,server-key}.pem /etc/docker/.tls/
```

Now that the credentials are in place, we need to create the `daemon.json` for the `docker.service`.

**NOTE:** *If the* `/etc/docker/daemon.json` *already exists, do not run the following* `cat` *command. Simply open your existing* `/etc/docker/daemon.json` *in your preferred editor and add the following* `json` *data to your existing* `/etc/docker/daemon.json`*.* 

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

Now, we restart the `docker.service`.

```bash
sudo systemctl restart docker
```

Next, from within your `~/.docker/tls` directory, test your connection with TLS.

```bash
docker --tlsverify \
--tlscacert=ca.pem \
--tlscert=cert.pem \
--tlskey=key.pem \
-H=$DOCKER_HOST_FQDN:2375 version
```

A successful test of your credentials should produce something like the following.

```bash
Client: Docker Engine - Community
 Version:           24.0.7
 API version:       1.43
 Go version:        go1.20.10
 Git commit:        afdd53b
 Built:             Thu Oct 26 09:07:41 2023
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          24.0.7
  API version:      1.43 (minimum version 1.12)
  Go version:       go1.20.10
  Git commit:       311b9ff
  Built:            Thu Oct 26 09:07:41 2023
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.26
  GitCommit:        3dd1e886e55dd695541fdcd67420c2888645a495
 runc:
  Version:          1.1.10
  GitCommit:        v1.1.10-0-g18a0cb0
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

**3. Finally, we can secure the client's connection by default.**

Copy your credentials to your `/home/.docker` directory.

```bash
cp -v {ca,cert,key}.pem ~/.docker
```

After which, you can do the following.

```bash
docker version # If this works, you're in the crypto!
```

**4. Add the environment variables to your `~/.docker/` directory and `~/.profile`.**

You TLS works now because the environment variables are still in memory, while will be flushed upon reboot. We can resolve this by creating a `.env_docker` file and loading it at login.

Create the `.env_docker` file and load it with the necessary environment variables.

```bash
cat > ~/.docker/.env_docker <<EOL
DOCKER_HOST="tcp://$DOCKER_HOST_FQDN:2375"
DOCKER_TLS_VERIFY=$DOCKER_TLS_VERIFY
EOL
```

Update your `.profile` to load the environment variables from `~/.docker/.env_docker`. We'll start with a blank new line so the new bash code isn't bunched up against the existing bash in the file.

```bash
cat >> ~/.profile <<'EOL'

# Load Docker TLS environment variables if '.env_docker' exists
if [ -f "$HOME/.docker/.env_docker" ]; then
    export $(cat $HOME/.docker/.env_docker | xargs)
fi
EOL
```

A quick reboot and you're ready to test your new environment.

**5. Endowing additional users with TLS for the secured Docker daemon socket.**

For any client that needs to connect to your newly secured Docker daemon socket, copy the contents of your `~/.docker/` directory into their `~/.docker` directory, update their `~/.profile` to load the `~/.docker/.env_docker`, and they're good as gold!

**Live long, and Docker!**

![Live long, and Docker!](https://media1.tenor.com/m/A3s-Mk6G2-kAAAAC/star-trek-spock.gif)

---

### Other Connection Modalities

You can read more about other options for connecting to a secure Docker daemon in the official [Docker Documentation](https://docs.docker.com/engine/security/protect-access/#other-modes).

---
