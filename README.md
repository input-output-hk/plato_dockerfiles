# Plato Image configuration:

Here you will find the steps in order to startup a plato node based on:
- Mantis client - https://github.com/input-output-hk/mantis
- CPU Ethminer - https://github.com/Genoil/cpp-ethereum/tree/108

## Setup a private network:

In order to setup Mantis image you need to provide a bind volume that contains a folder called `conf` with all the custom configuration related to Mantis.

_Note: Refer to [Mantis](http://mantis.readthedocs.io/en/latest/General-Configuration/) documentation in order to understand configuration parameters_

- Create a file called `private-genesis.json` into the folder `conf`, with a configuration like this: (In order to allow nodes
to connected between them, they need to have this file with the same configuration)
```
{
  "extraData": "0x00",
  "nonce": "0x0000000000000042",
  "gasLimit": "0x2fefd8",
  "difficulty": "0x400",
  "ommersHash": "0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347",
  "timestamp": "0x00",
  "coinbase": "0x0000000000000000000000000000000000000000",
  "mixHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "alloc": {}
}
```

- In the existing file `network.conf` set the boostrap-nodes: (In order to connect one node
with another, at least one of them, needs to config the other as a boostrap-node)
```
mantis {
  network {
    ...

    discovery {
      # Set of initial nodes
      bootstrap-nodes = [
        "enode://$NODE1_ID@$NODE1_IP:NODE1_PORT"
        "enode://$NODE2_ID@$NODE2_IP:NODE2_PORT"
      ]
    }

    ...
  }
}
```
Replace the variables $NODE1_ID, $NODE1_IP, $NODE1_PORT, etc with the values that correspond.

After this configuration, you can build the images using the command below:
``` 
$ export MANTIS_VERSION="0.3-cli-beta"; export RPC_PORT=8545; export MANTIS_CONF=./conf; export DATADIR=./db-data; docker-compose up --build -d mantis
```
As you can see from the command, you need to setup a couple of env variables:
- MANTIS_VERSION: The [released version of mantis](https://github.com/input-output-hk/mantis/releases/tag/v0.3-cli-beta) (image used for download client)
- RPC_PORT: The port that the node is going to expose
- MANTIS_CONF: The path where the folder with the configuration files is located (you can get the initial files from a [distribution package](https://github.com/input-output-hk/mantis/releases/download/v0.3-cli-beta/mantis-0.3-cli-beta.zip), it is highly recommend that
the initial config files match the MANTIS_VERSION you previously defined). This is managed by the docker-compose as a binded volume.
- DATADIR: A path to the folder that will contains the datadir, this is manage by the docker-compose as a binded volume too.

## Configuring the miner:

After building the mantis image, you need to create an account for the miner. You can use a command like this: 
```
curl -X POST \
  http://127.0.0.1:$RPC_PORT/ \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -H 'postman-token: 9b445681-1280-ccec-440f-ac5838df4686' \
  -d '{
    "jsonrpc": "2.0",
    "method": "personal_newAccount", 
    "params": ["$ACCOUNT_PASSWORD"],
    "id": 67
}'
```
Replace $RPC_PORT with the port you used after, and $ACCOUNT_PASSWORD with a secure secret password.
The expected output is:
```
{"jsonrpc":"2.0","result":"$NEW_ADDRESS","id":67}
```

Then configure the files related to the miner.

Highlight configuration files for setup for the ethminer:

- In the existing file `sync.conf` set on regular sync.
```
mantis {
  sync {
    # Whether to enable fast-sync
    do-fast-sync = false
    
    ...
  }
}
```

- In the existing file `misc.conf`
```
mantis {
  ...

  mining {
    # Miner's coinbase address
    coinbase = "$MINER_ADDRESS"
  }

  ...
}
```
Replace $MINER_ADDRESS with the $NEW_ADDRESS created before.

Now you can restart the service and then start the miner again:
```
$ docker-compose restart mantis

$ export MANTIS_VERSION="0.3-cli-beta"; export RPC_PORT=8545; export MANTIS_CONF=./conf; export DATADIR=./db-data; docker-compose up --build -d ethminer
```

# Know issues:

- If you are using a hosting service, be sure that you have enough cpu power to do mining (If using Amazon EC2, computing oriented ones are suggested)

- If the miner takes too much time to start mining, follow this steps:
```
$ docker exec -it plato_ethminer_1 bash
$ (container) ./ethminer -D 0
```
You should see `Initializing DAG for epoch beginning #0 (seedhash 00000000…). This will take a while.`

Once that's finished, restart ethminer container:
```
export MANTIS_VERSION="0.3-cli-beta"; export RPC_PORT=8545; export MANTIS_CONF=./conf; export DATADIR=./db-data ; docker-compose restart ethminer
```

- If you are in a server connected by SSH and you close your session, when you re-connect, you must run the docker-compose commands re-exporting the env variables previously defined.