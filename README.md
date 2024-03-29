# setting up indy network

## reqiuired things
   * indy node 
   * utility image (ubuntu 16 based docker image which has indy node and indy-cli installed)

## indy node

i took a reference from indy node container repo https://github.com/hyperledger/indy-node-container

and from the build folder took the Dockerfile.ubuntu20 and made some changes to look like this

``` 
FROM ubuntu:20.04

RUN apt-get update -y && apt-get install -y \
    apt-transport-https \
    ca-certificates \
    gnupg2 \
    ## ToDo remove unused packages
    libgflags-dev \
    libsnappy-dev \
    zlib1g-dev \
    libbz2-dev \
    liblz4-dev \
    libgflags-dev \
    python3-pip

# Bionic-security for libssl1.0.0
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3B4FE6ACC0B21F32 \
    && echo "deb http://security.ubuntu.com/ubuntu bionic-security main"  >> /etc/apt/sources.list

# Sovrin
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys CE7709D068DB5E88 \
    && bash -c 'echo "deb https://repo.sovrin.org/deb bionic master" >> /etc/apt/sources.list'

# Hyperledger Artifactory
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 9692C00E657DDE61 \
    && echo "deb https://hyperledger.jfrog.io/artifactory/indy focal rc" >> /etc/apt/sources.list \
    # Prioritize packages from hyperledger.jfrog.io
    && printf '%s\n%s\n%s\n' 'Package: *' 'Pin: origin hyperledger.jfrog.io' 'Pin-Priority: 1001' >> /etc/apt/preferences

RUN pip3 install -U \
    # Required by setup.py
    'setuptools==50.3.2'


RUN apt-get update -y && apt-get install -y \
    indy-node="1.13.2~rc6" \
    && rm -rf /var/lib/apt/lists/* \
    # fix path to libursa
    && ln -s /usr/lib/ursa/libursa.so /usr/lib/libursa.so
RUN mkdir -p /var/lib/indypvcvolume/
COPY indy_config.py /etc/indy/

WORKDIR /home/indy
```

it helped me creatin a docker image for indy node 

## utility image

so again i took refernce from indy-node-container https://github.com/hyperledger/indy-node-container and official docs https://hyperledger-indy.readthedocs.io/projects/sdk/en/latest/cli/README.html

``` 
FROM ubuntu:16.04

RUN apt-get update

RUN apt-get install -y curl \
    && curl -sL https://deb.nodesource.com/setup_14.x | bash - \
    && apt-get install -y nodejs

RUN apt-get update -y && apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    wget \
    xz-utils \
    && rm -rf /var/lib/apt/lists/*



RUN cd /home/

RUN apt-get update && \
    apt-get install -y \
    tmux \
    && rm -rf /var/lib/apt/lists/*

RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys CE7709D068DB5E88
RUN bash -c 'echo "deb https://repo.sovrin.org/deb xenial stable" >> /etc/apt/sources.list'

RUN apt-get update -y && apt-get install -y \
    indy-node=1.12.6 \
    indy-plenum=1.12.6 \
    && rm -rf /var/lib/apt/lists/*

RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys CE7709D068DB5E88
RUN bash -c 'echo "deb https://repo.sovrin.org/sdk/deb xenial stable" >> /etc/apt/sources.list'
RUN apt-get update && apt-get install -y indy-cli
RUN rm -rf /var/lib/apt/lists/*

COPY latestStableScript /home/indy/
COPY indy_config.py /etc/indy/indy_config.py

WORKDIR /home/indy/indy-server
RUN chmod 777 -R ../*
ENTRYPOINT ["bash", "-c", "node server.js"]
```

and written some custom shell script and javascript to which utilizes the below written command, indy-cli and indy-node installed on utility to generate steward and trustees csv files.

so these command were used to create dids and keys yequired for trustees and stewards csv files

```
wallet create newNetwork key=key
wallet open newNetwork key=key
did new seed=000000000000000000000000Trustee1
did new seed=000000000000000000000000Trustee2
did new seed=000000000000000000000000Trustee3
did new seed=000000000000000000000000Steward1
did new seed=000000000000000000000000Steward2
did new seed=000000000000000000000000Steward3
did new seed=000000000000000000000000Steward4
```

and other node related keys (verkeys, bls, and bls pop) keys were created using these command 

``` 
init_indy_keys --name node1 --seed $STEWARD_SEED1
init_indy_keys --name node2 --seed $STEWARD_SEED2 
init_indy_keys --name node3 --seed $STEWARD_SEED3 -
init_indy_keys --name node4 --seed $STEWARD_SEED4
```


these csv files are then utilized by 'create_genesis.py' from indy node to generate genesis data.


then this genesis data and keys are utlized by the indy node to start running.

command used to start indy_node 

```
start_indy_node node1 0.0.0.0 9701 0.0.0.0 9702
start_indy_node node2 0.0.0.0 9703 0.0.0.0 9704
start_indy_node node3 0.0.0.0 9705 0.0.0.0 9706
start_indy_node node4 0.0.0.0 9707 0.0.0.0 9708
```
