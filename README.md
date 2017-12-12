# Plato Image configuration:

Here you will find the steps in order to startup a Plato node based on:
- Plato client (Mantis fork) - https://github.com/input-output-hk/plato

## Setup a private network:

In order to setup Plato image you need to provide a bind volume that contains a folder called `conf` with all the custom configuration related to Plato private network.

_Note: Refer to [Plato](https://github.com/input-output-hk/plato/wiki) documentation in order to understand configuration parameters_

- Create a file called `private-genesis.json` into the folder `conf`, with a configuration like this: (In order to allow nodes
to connected between them, they need to have this file with the same configuration)
```
{
  "timestamp": "0x5a17174d",
  "difficulty": "0x0400000000",
  "gasLimit": "0x1388000000",
  "nonce": "0x0000000000000042",
  "extraData": "0x11bbe8db4e347b4e8c937c1c8370e4b5ed33adb3db69cbdb7a38e1e50b1b82fa",
  "ommersHash": "0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347",
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
 
- In the existing file `ouroboros.conf` set the the accounts that will be part of the consensus algorithm:
```
mantis {
  ouroboros {
    slot-minerStakeHolders-mapping =
      {
        1:  ["edde8656c35fcb7126c61fc6e2673734425a72bf","240653414e3d40764a0fc75af372bb6458f8600a"]
      }
  }
}
```
In the example we configure the wallets `0xedde8656c35fcb7126c61fc6e2673734425a72bf` and `0x240653414e3d40764a0fc75af372bb6458f8600a` as the set of authorities that can add new blocks from slot 1. For more info, check the Plato documentation section [Configuring Certification Authorities](https://github.com/input-output-hk/plato/wiki/Configuring-Certification-Authorities-(CAs))

## Build your Plato image

After this configuration, you can build the images using the command below:
``` 
$ export MANTIS_VERSION="0.6-cardano-enterprise-node"; export RPC_PORT=8545; export MANTIS_CONF=./conf; export DATADIR=./db-data; docker-compose up --build -d mantis

```
As you can see from the command, you need to setup a couple of env variables:
- MANTIS_VERSION: The [released version of plato](https://github.com/input-output-hk/plato/releases/tag/v0.6-cardano-enterprise-node)
- RPC_PORT: The port that the node is going to expose
- MANTIS_CONF: The path where the folder with the configuration files is located (you can get the initial files from a [distribution package](https://github.com/input-output-hk/plato/archive/v0.6-cardano-enterprise-node.zip), it is highly recommend that
the initial config files match the MANTIS_VERSION you previously defined). This is managed by the docker-compose as a binded volume.
- DATADIR: A path to the folder that will contains the datadir, this is manage by the docker-compose as a binded volume too.

## Add a new certification authority to the node:

If you want to add a certification authority. Previously you should have an account managed by your node. If you don't have an account you could use a command like this to create it: 
```
curl -X POST \
  http://$NODE_IP:$RPC_PORT/ \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -d '{
    "jsonrpc": "2.0",
    "method": "personal_newAccount", 
    "params": ["$ACCOUNT_PASSWORD"],
    "id": 67
}'
```
Replace $NODE_IP with the ip of the node you want to managed the account and $RPC_PORT with the port you used after, and $ACCOUNT_PASSWORD with a secure secret password.
The expected output is:
```
{"jsonrpc":"2.0","result":"$NEW_ADDRESS","id":67}
```

Then configure the file related to the certification authorities consensus. Please follow the documentation guide:
[Configuring initial CAs](https://github.com/input-output-hk/plato/wiki/Configuring-Certification-Authorities-(CAs)#configuring-initial-cas)

IMPORTANT: Take in count that any new authority implies a change in the current consensus algorithm (this is consider a hard fork), so in order to maintain consesus between all the network nodes, each node should configure the new certification authority in its settings, and be restarted in order to apply changes.
After update the settings file you can restart the node:
```
export MANTIS_VERSION="0.6-cardano-enterprise-node"; export RPC_PORT=8545; export MANTIS_CONF=./conf; export DATADIR=./db-data; docker-compose restart mantis
```
Unlock the new certification authority address:
```
curl -X POST \
  "$NODE_IP:$NODE_JSON_RPC_PORT"/ \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -d '{ "jsonrpc": "2.0", "method": "personal_unlockMinerAccount", "params": ["$ADDRESS", "$PASSWORD"], "id": 1}'
```

- $NODE_IP:$NODE_JSON_RPC_PORT = Node IP and JSON-RPC port from the node that manage the account.

- $ADDRESS = Authority wallet address (without 0x prefix)

- $PASSWORD = *** (replace with your wallet password)

# Troubleshooting:

## Locked certification authority

In order to check, using logs, if some of your certification authorities are locked you should follow the steps below:

- First if you are using a hosting service, enter via ssh to the instance.

- Then execute the following command (pre-condition: Plato docker image should be running):

$ docker exec -it platoiohk_mantis_1 bash

- We can check if the there are some authority locked with the following command:

$ grep "Unable to get block for mining, error: Error while generating block for mining: LockedMinerAccountError" mantis.log

If you execute the command several times and the output constantly grows, it's probably some of the node certification authorities are locked.

- In order to unlock them, follow the previous guide [Unlocking CA accounts](https://github.com/input-output-hk/plato/wiki/Configuring-Certification-Authorities-(CAs)#unlocking-ca-accounts)


