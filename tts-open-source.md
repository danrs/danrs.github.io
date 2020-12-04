Installing The Things Stack open source with custom certificates is a complex task with quite a few configuration settings. As of November 2020, the [official guide](https://thethingsstack.io/getting-started/installation/) does not describe all the steps needed, so this page will serve as a complete guide.

# Prerequisites
- A server with a recommended 4 virtual CPUs and 16GB RAM running Docker and Docker Compose*
- DNS record pointing to your computer's IP address. If hosting privately, you can just use the private IP address (this probably looks like 10.x.x.x or 192.168.x.x).
- Install [docker](https://docs.docker.com/engine/install/ubuntu/) and [docker-compose](https://docs.docker.com/compose/install/#install-compose)

```
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER

sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

- Install go

```
sudo snap install go --classic
```

- Install `cfssl` and `cfssljson`:

```
go get -u github.com/cloudflare/cfssl/cmd/cfssl
go get -u github.com/cloudflare/cfssl/cmd/cfssljson
```


*Benchmark for 100K devices with 12 confirmed uplinks per day. Your requirements will vary depending on your load and desired redundancy. When testing the installation process (so 0 devices and 0 uploads) I got it running on a 4-core laptop with 4GB of RAM.

# Configuration
- Download [ttn-lw-stack-docker.yml](https://thethingsstack.io/getting-started/installation/configuration/ttn-lw-stack-docker-open-source.yml) and [docker-compose.yml](https://thethingsstack.io/getting-started/installation/configuration/docker-compose-open-source.yml). Note that `.yml != .yaml`, so don't get smart and change the file extension because it will throw something off somewhere.
- Create a directory structure. I'm putting everything in a directory called `tts`

```
mkdir tts
mkdir -p tts/config/stack
mv ttn-lw-stack-docker-open-source.yml tts/config/stack/ttn-lw-stack-docker.yml
mv docker-compose-open-source.yml tts/docker-compose.yml
```

If you run `tree tts` you should see the following:

```
tts
├── config
│   └── stack
│       └── ttn-lw-stack-docker.yml
└── docker-compose.yml
```

- Configure your Docker containers by editing `docker-compose.yml`.
  - Replace all instances of `thethings.example.com` with the IP address of your host machine. You can leave `127.0.0.1` in the redis, postgres and CockroachDB sections.
  - If using postgres, comment out the Cockroachdb stuff and uncomment the postgres stuff (there are 3 places: container definition, stack dependency, and a stack env variable)
  - Uncomment the sections on secrets at the bottom of the file
  - Add additional environment variables as described in [this github issue](https://github.com/TheThingsNetwork/lorawan-stack/issues/1230). Note that some of the variables mentioned in that issue overlap with config settings in `ttn-lw-stack-docker.yml`, so I have not repeated them here. The environment variables I have set are as follows: 
  
```
    environment:
      TTN_LW_BLOB_LOCAL_DIRECTORY: /srv/ttn-lorawan/public/blob
      TTN_LW_REDIS_ADDRESS: redis:6379
      # If using CockroachDB:
      # TTN_LW_IS_DATABASE_URI: postgres://root@cockroach:26257/ttn_lorawan?sslmode=disable
      # # If using PostgreSQL:
      TTN_LW_IS_DATABASE_URI: postgres://root:root@postgres:5432/ttn_lorawan?sslmode=disable

      TTN_LW_APPLICATION_SERVER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_DEVICE_TEMPLATE_CONVERTER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_GATEWAY_SERVER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_IDENTITY_SERVER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_JOIN_SERVER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_NETWORK_SERVER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_OAUTH_SERVER_ADDRESS: https://192.168.x.x:8885/oauth
```

- Configure The Things Stack by editing `config/stack/ttn-lw-stack-docker.yml`.
  - Replace all instances of `thethings.example.com` with the IP address of your host machine.
  - Uncomment the custom certs and comment out let's encrypt
  - Add port `8885` to most the urls specified below. There may be other urls which require ports to be specified (such as those for `dcs`).
    - `is.email.network.console-url`
    - `is.email.network.identity-server-url`
    - Every url nested under `console` 
  - Change the console secret to a new value (optional)
- Compare your configured files against the sample files listed at the bottom of this page
- See the [official guide](https://thethingsstack.io/getting-started/installation/configuration/) for further configuration if necessary.
- If you are not using https, replace all instances of https in the configuration files with http. You will also need to change any port numbers that you've specified in URLs from `888x` to `188x`.

# Certificates from a Custom Certificate Authority
If you are not using https, you can skip this step. As of 4 Dec 2020, I have not yet tested if the system works fully without https, but it should be possible. Note that there are additional configuration instructions in this guide that you must follow if you are not using https.

- Create `ca.json`:

```
{
  "names": [
    {"C": "NL", "ST": "Noord-Holland", "L": "Amsterdam", "O": "The Things Demo"}
  ]
}
```

- Create `cert.json`, using your host machine IP:

```
{
  "hosts": ["192.168.x.x"],
  "names": [
    {"C": "NL", "ST": "Noord-Holland", "L": "Amsterdam", "O": "The Things Demo"}
  ]
}
```

- Generate cert and cert authority and add the relevant files to your project:

```
cfssl genkey -initca ca.json | cfssljson -bare ca
cfssl gencert -ca ca.pem -ca-key ca-key.pem cert.json | cfssljson -bare cert
mv cert-key.pem tts/key.pem
mv cert.pem tts/cert.pem
mv ca.pem tts/ca.pem
```

- You can remove any other generated files (`ca-key.pem, ca.csr, ca.json, cert.csr, cert.json`); you don't need them. Your project directory should look like this:

```
tts
├── ca.pem
├── cert.pem
├── config
│   └── stack
│       └── ttn-lw-stack-docker.yml
├── docker-compose.yml
└── key.pem
```

- Add `ca.pem` to the cert store on any machines that will be accessing the stack. Note that browsers often have their own cert stores, and the cert store location will of course be different on windows. You will also need to add the cert authority to your actual stack container once it is running.

```
sudo cp ca.pem /usr/local/share/ca-certificates/ca.crt
sudo update-ca-certificates
```

# Running The Things Stack
- Pull docker images

```
docker-compose pull
```

- Initialise db

```
docker-compose run --rm stack is-db init
```

- Create an admin user

```
docker-compose run --rm stack is-db create-admin-user \
  --id admin \
  --email your@email.com --password password
```

- Register command line and web console with oauth. If you are not using https, change these urls to `http` and replace port `8885` with `1885`.

```
docker-compose run --rm stack is-db create-oauth-client \
  --id cli \
  --name "Command Line Interface" \
  --owner admin \
  --no-secret \
  --redirect-uri "local-callback" \
  --redirect-uri "code"

docker-compose run --rm stack is-db create-oauth-client \
  --id console \
  --name "Console" \
  --owner admin \
  --secret console # use the console secret specified in ttn-lw-stack-docker.yml \
  --redirect-uri "https://192.168.x.x/console/oauth/callback" \
  --redirect-uri "/console/oauth/callback" \
  --logout-redirect-uri "https://192.168.x.x/console" \
  --logout-redirect-uri "/console"
```

- Launch

```
docker-compose up -d
```

# Using the Stack
- As promised, we are going to copy the cert auth into our stack. Yes, this could be done automatically by mounting the file like the we did for the config yaml but I've not worked out how to do that.

```
cat ca.pem # copy the contents into your cliboard or something
docker exec -it -u 0 tts_stack_1 /bin/sh # shell into the container as root
echo 'PASTE_CERT_CONTENTS_HERE' > /usr/local/share/ca-certificates/ca.crt
update-ca-certificates
```

- We can now log in and use the stack

```
ttn-lw-cli login --callback=false # as prompted, open your browser and paste the token back into the terminal
```

# Stopping the stack
If you don't have other docker processes running, you can just blow everything away with the commands below. If you have other processes running you probably know how to handle them.

```
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
```

# Appendix: Sample yaml files
## docker-compose.yml

```
version: '3.7'
services:

  # If using CockroachDB:
  #  cockroach:
  #  # In production, replace 'latest' with tag from https://hub.docker.com/r/cockroachdb/cockroach/tags
  #  image: cockroachdb/cockroach:latest
  #  command: start-single-node --http-port 26256 --insecure
  #  restart: unless-stopped
  #  volumes:
  #    - ${DEV_DATA_DIR:-.env/data}/cockroach:/cockroach/cockroach-data
  #  ports:
  #    - "127.0.0.1:26257:26257" # Cockroach
  #    - "127.0.0.1:26256:26256" # WebUI

  # If using PostgreSQL:
  postgres:
    image: postgres
    restart: unless-stopped
    environment:
      - POSTGRES_PASSWORD=root
      - POSTGRES_USER=root
      - POSTGRES_DB=ttn_lorawan
    volumes:
      - ${DEV_DATA_DIR:-.env/data}/postgres:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"

  redis:
    # In production, replace 'latest' with tag from https://hub.docker.com/_/redis?tab=tags
    image: redis:latest
    command: redis-server --appendonly yes
    restart: unless-stopped
    volumes:
      - ${DEV_DATA_DIR:-.env/data}/redis:/data
    ports:
      - "127.0.0.1:6379:6379"

  stack:
    # In production, replace 'latest' with tag from https://hub.docker.com/r/thethingsnetwork/lorawan-stack/tags
    image: thethingsnetwork/lorawan-stack:latest
    entrypoint: ttn-lw-stack -c /config/ttn-lw-stack-docker.yml
    command: start
    restart: unless-stopped
    depends_on:
      - redis
      # If using CockroachDB:
      # - cockroach
      # If using PostgreSQL:
      - postgres
    volumes:
      - ./blob:/srv/ttn-lorawan/public/blob
      - ./config/stack:/config:ro
      # If using Let's Encrypt:
      # - ./acme:/var/lib/acme
    environment:
      TTN_LW_BLOB_LOCAL_DIRECTORY: /srv/ttn-lorawan/public/blob
      TTN_LW_REDIS_ADDRESS: redis:6379
      # If using CockroachDB:
      # TTN_LW_IS_DATABASE_URI: postgres://root@cockroach:26257/ttn_lorawan?sslmode=disable
      # # If using PostgreSQL:
      TTN_LW_IS_DATABASE_URI: postgres://root:root@postgres:5432/ttn_lorawan?sslmode=disable

      TTN_LW_APPLICATION_SERVER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_DEVICE_TEMPLATE_CONVERTER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_GATEWAY_SERVER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_IDENTITY_SERVER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_JOIN_SERVER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_NETWORK_SERVER_GRPC_ADDRESS: 192.168.x.x:8884
      TTN_LW_OAUTH_SERVER_ADDRESS: https://192.168.x.x:8885/oauth

    ports:
      # If deploying on a public server:
      - "80:1885"
      - "443:8885"
      - "1881:1881"
      - "8881:8881"
      - "1882:1882"
      - "8882:8882"
      - "1883:1883"
      - "8883:8883"
      - "1884:1884"
      - "8884:8884"
      - "1885:1885"
      - "8885:8885"
      - "1887:1887"
      - "8887:8887"
      - "1700:1700/udp"

    # If using custom certificates:
    secrets:
      - ca.pem
      - cert.pem
      - key.pem

# If using custom certificates:
secrets:
  ca.pem:
    file: ./ca.pem
  cert.pem:
    file: ./cert.pem
  key.pem:
    file: ./key.pem
```

## ttn-lw-stack-docker.yml

```
# Identity Server configuration
# Email configuration for "192.168.x.x"
is:
  email:
    sender-name: 'The Things Stack'
    sender-address: 'noreply@192.168.x.x'
    network:
      name: 'The Things Stack'
      console-url: 'https://192.168.x.x:8885/console'
      identity-server-url: 'https://192.168.x.x:8885/oauth'

    # If sending email with Sendgrid
    # provider: sendgrid
    # sendgrid:
    #   api-key: '...'              # enter Sendgrid API key

    # If sending email with SMTP
    # provider: smtp
    # smtp:
    #   address:  '...'             # enter SMTP server address
    #   username: '...'             # enter SMTP server username
    #   password: '...'             # enter SMTP server password

  # Web UI configuration for "192.168.x.x":
  oauth:
    ui:
      canonical-url: 'https://192.168.x.x/oauth'
      is:
        base-url: 'https://192.168.x.x/api/v3'

# HTTP server configuration
http:
  cookie:
    block-key: ''                # generate 32 bytes (openssl rand -hex 32)
    hash-key: ''                 # generate 64 bytes (openssl rand -hex 64)
  metrics:
    password: 'metrics'               # choose a password
  pprof:
    password: 'pprof'                 # choose a password

# If using custom certificates:
tls:
  source: file
  root-ca: /run/secrets/ca.pem
  certificate: /run/secrets/cert.pem
  key: /run/secrets/key.pem

# Let's encrypt for "192.168.x.x"
#tls:
#  source: 'acme'
#  acme:
#    dir: '/var/lib/acme'
#    email: 'you@192.168.x.x'
#    hosts: ['192.168.x.x']
#    default-host: '192.168.x.x'

# If Gateway Server enabled, defaults for "192.168.x.x":
gs:
  mqtt:
    public-address: '192.168.x.x:1882'
    public-tls-address: '192.168.x.x:8882'
  mqtt-v2:
    public-address: '192.168.x.x:1881'
    public-tls-address: '192.168.x.x:8881'

# If Gateway Configuration Server enabled, defaults for "192.168.x.x":
gcs:
  basic-station:
    default:
      lns-uri: 'wss://192.168.x.x:8887'
  the-things-gateway:
    default:
      mqtt-server: 'mqtts://192.168.x.x:8881'

# Web UI configuration for "192.168.x.x":
console:
  ui:
    canonical-url: 'https://192.168.x.x:8885/console'
    is:
      base-url: 'https://192.168.x.x:8885/api/v3'
    gs:
      base-url: 'https://192.168.x.x:8885/api/v3'
    ns:
      base-url: 'https://192.168.x.x:8885/api/v3'
    as:
      base-url: 'https://192.168.x.x:8885/api/v3'
    js:
      base-url: 'https://192.168.x.x:8885/api/v3'
    qrg:
      base-url: 'https://192.168.x.x:8885/api/v3'
    edtc:
      base-url: 'https://192.168.x.x:8885/api/v3'

  oauth:
    authorize-url: 'https://192.168.x.x:8885/oauth/authorize'
    token-url: 'https://192.168.x.x:8885/oauth/token'
    logout-url: 'https://192.168.x.x:8885/oauth/logout'
    client-id: 'console'
    client-secret: 'console'          # choose or generate a secret

# If Application Server enabled, defaults for "192.168.x.x":
as:
  mqtt:
    public-address: 'https://192.168.x.x:1883'
    public-tls-address: 'https://192.168.x.x:8883'
  webhooks:
    downlink:
      public-address: '192.168.x.x:1885/api/v3'

# If Device Claiming Server enabled, defaults for "192.168.x.x":
dcs:
  oauth:
    authorize-url: 'https://192.168.x.x/oauth/authorize'
    token-url: 'https://192.168.x.x/oauth/token'
    logout-url: 'https://192.168.x.x/oauth/logout'
    client-id: 'device-claiming'
    client-secret: 'device-claiming'          # choose or generate a secret
  ui:
    canonical-url: 'https://192.168.x.x/claim'
    as:
      base-url: 'https://192.168.x.x/api/v3'
    dcs:
      base-url: 'https://192.168.x.x/api/v3'
    is:
      base-url: 'https://192.168.x.x/api/v3'
    ns:
      base-url: 'https://192.168.x.x/api/v3'
```




