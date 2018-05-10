# Factomd Control Plane

This repository is responsible for maintaining the control plane for factomd M3.

It includes 4 containers:
  1. FactomD
  > Runs the factom node
  2. SSH
  > Permits ssh access only to _this specific container_. Mounts the factomd database volume for debugging purposes.
  3. Filebeat
  > Reports stdout/stderr of all docker containers to our elasticsearch instance
  4. Metricbeat
  > Reports hardware metrics of all docker containers to our elasticsearch instance.

# Install docker

Please follow the instructions [here](https://docs.docker.com/install/linux/docker-ce/ubuntu/) to install docker-ce to your machine.

Then, run `usermod -aG docker $USER` and logout/login.


# Configure Docker

In order to join the swarm, first ensure that your firewall rules allow access on the following ports. All swarm communications occur over a self-signed TLS certificate.

- TCP port `2376` _only to_ `52.48.130.243` for secure Docker engine communication. This port is required for Docker Machine to work. Docker Machine is used to orchestrate Docker hosts.

In addition,  the following ports must be opened for factomd to function:
- `2222` to `52.48.130.243`, which is the SSH port used by the `ssh` container
- `8088` to `52.48.130.243`, the factomd API port
- `8108` to the world, the factomd mainnet port

An example using `iptables`:
```
sudo iptables -A INPUT -p tcp -s 52.48.130.243 --dport 2376 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp -s 52.48.130.243 --dport 2222 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp -s 52.48.130.243 --dport 8088 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 8108 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo service iptables save
```

# Exposing the Docker Engine

### Using `daemon.json` (recommended)

You can configure the docker daemon using a default config file, located at `/etc/docker/daemon.json`. Create this file if it does not exist.

Example configuration:
```
{
  "tls": true,
  "tlscert": "/path/to/cert.pem"
  "tlskey": "/path/to/key.pem",
  "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock"]
}
```

### On RedHat

Open (using `sudo`) `/etc/sysconfig/docker` in your favorite text editor.

Append `-H=unix:///var/run/docker.sock -H=0.0.0.0:2376 --tls --tlscert=<path to cert.pem> --tlskey=<path to key.pem>` to the pre-existing OPTIONS

Then, `sudo service docker restart`.

### Using `systemd`

Run `sudo systemctl edit docker.service`

Edit the file to match this:

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2376 --tls --tlscert <path to cert.pem> --tlskey <path to key.pem>
```

Then reload the configuration:
`sudo systemctl daemon-reload`

and restart docker:

`sudo systemctl restart docker.service`

### I don't want to use a process manager

You can manually start the docker daemon via:

```sudo dockerd -H=unix:///var/run/docker.sock -H=0.0.0.0:2376 --tlscert=<path to cert.pem> --tlskey=<path to key.pem>```

# Create the FactomD volumes
Factomd relies on two volumes,`factom_database` and `factom_keys`. Please create these before joining the swarm.

1. `docker volume create factom_database`
2. `docker volume create factom_keys`

These volumes will be used to store information by the `factomd` container.

If you already have a synced node and would like to avoid resyncing, run:

`sudo cp -r <path to your database> /var/lib/docker/volumes/factom_database/_data`.

In addition, please place your `factomd.conf` file in `/var/lib/docker/volumes/factom_keys/_data`.

# Join the Docker Swarm

Finally, to join the swarm:
```
docker swarm join --token SWMTKN-1-5ct5plmbn1ombbjqp8ql8hq93jkof6246suzast5n1gfwa083b-1ui6w6fupe45tizz0tv6syzrs 52.48.130.243:2377

```

As a reminder, joining as a worker means you have no ability to control containers on another node.

Once you have joined the network, you will be issued a control panel login by a Factom employee.

**Only accept logins at federation.factomd.com. Any other login endpoints are fraudulent and not to be trusted.**

# Starting FactomD Container

There are two means of launching your `factomd` instance:

### From the Docker CLI

Run this command _exactly_: `docker run -d --name "factomd" -v "factom_database:/root/.factom/m2" -v "factom_keys:/root/.factom/private" -p "8088:8088" -p "8090:8090" -p "8108:8108" -l "name=factomd" factominc/factomd:v5.0.0-alpine -startdelay=600 -config=/root/.factom/private/factomd.conf
`

### From the Portainer UI

Once you have logged into the [control panel](https://federation.factomd.com), please ensure your node is selected in the top left dropdown menu.

Then, click `containers > add container`.

**:heavy_exclamation_mark: These instructions must be followed exactly, otherwise you risk being kicked from the authority set. :heavy_exclamation_mark:**

1. Name your container `factomd`.

2. Enter the image name `factominc/factomd:v5.0.0-alpine`

3. Mark additional ports `8088:8088`, `8108`:`8108`, `8090:8090`.

4. Do _not_ modify access control.

5. Either this command for the command:  `-startdelay=600 -config=/root/.factom/private/factomd.conf`or your own flags. But be careful!

6. Click "volumes", and map `/root/.factom/m2` to `factom_database`, and `/root/.factom/private` to `factom_keys`.

7. Click "labels" and add a label `name:name` = `value:factomd`

8. Click "deploy the container"

9. You are done!


### NOTE: The Swarm cluster is still experimental, so please pardon our dust! If you have an issues, please contact ian at factom dot com.
