# Exposition of *setup.sh*

The *setup.sh* script creates a basic Quorum network with the genesis file configured for Istanbul consensus.

This is what we set up for each node.

 * Ether account and *keystore* directory for each node.
   * The account gets written into the *genesis.json* file that each node runs once to bootstrap the blockchain.
   * **YET the genesis.json still needs to be modified before bootstrapping the node!**
 * The *tm.conf* file that tells Quorum where all the node's keys are and where all the other nodes are.
 * Public/private Keypairs for Quorum private transactions.
 * A script for starting the Geth and Constellation processes in each container, *start-node.sh*.
 * A folder, *logs/*, for Geth and Constellation to write their log files to.

In addition we create some utility scripts on the host.

  * A *docker-compose.yml* file that can be used with docker-compose to create the network of containers.

Refer to the *setup.sh* file itself for the full code.

## Configuration options

Options are simple and self-explanatory. The *docker-compose.yml* file will create a Docker network for the nodes as per the `subnet` variable here. If you want to run more nodes, then add addresses for them to the `ips` list.

    #### Configuration options #############################################

    # One Docker container will be configured for each IP address in $ips
    subnet="172.13.0.0/16"
    ips=("172.13.0.2" "172.13.0.3" "172.13.0.4")

    # Docker image name
    image=quorum

The docker image is used during set-up to run Geth, Bootnode and Constellation to generate various things. These executables don't need to be installed on the host machine.

## House-keeping

At least two nodes have to be configured in the current version. 

    if [[ ${#ips[@]} < 2 ]]
    then
        echo "ERROR: There must be more than one node IP address."
    exit 1
    fi

Delete any old configuration.

    ./cleanup.sh

We will need to run processes within the Docker containers with the same account parameters as the user on the Docker host. This is to avoid problems with the mapped disk volumes that are shared between the host and the containers. So we collect the info here for later use.

    uid=`id -u`
    gid=`id -g`
    pwd=`pwd`

## Directory structure

The final goal at the end of set-up is for each node to have its own directory tree that looks like this:

    /qdata/
    ├── dd/
    │   ├── geth/
    │   ├── keystore/
    │   │   └── UTC--2017-10-21T12-49-26.422099203Z--aad5479aff498c9258b21b59dd7546262aa2cfc7
    │   ├── nodekey
    │   └── static-nodes.json
    ├── keys/
    │   ├── tma.key
    │   ├── tma.pub
    │   ├── tm.key
    │   └── tm.pub
    ├── logs/
    ├── genesis.json
    ├── passwords.txt
    ├── start-node.sh
    └── tm.conf

On the Docker host, we create a *qdata_N/* directory for each node, with this structure. When we start up the network, this will be mapped by the *docker-compose.yml* file to each container's internal */qdata/* directory.

    #### Create directories for each node's configuration ##################

    n=1
    for ip in ${ips[*]}
    do
        qd=qdata_$n
        mkdir -p $qd/{logs,keys}
        mkdir -p $qd/dd/geth

        let n++
    done

## Create Enode information and *static-nodes.json*

**This part is not supported with our iteration of this process. The enode information currently has to be gathered from up-and-running nodes, since the upgrade from Quorum 1.2.0 to Quorum 2.0.2 broke this process.**

*Each node is assigned an Enode, which is the public key corresponding to a private *nodekey*. This Enode is what identifies the node on the Ethereum network. Membership of our private network is defined by the Enodes listed in the *static-nodes.json* file. These are the nodes that can participate in the Raft consensus.

We use Geth's *bootnode* utility to generate the Enode and the private key. By jumping through some hoops to get the file permissions right we can use the version of *bootnode* already present in the Docker image.*

    #### Make static-nodes.json and store keys #############################

    echo "[" > static-nodes.json
    n=1
    for ip in ${ips[*]}
    do
        qd=qdata_$n

        # Generate the node's Enode and key
        enode=`docker run -u $uid:$gid -v $pwd/$qd:/qdata $image /usr/local/bin/bootnode -genkey /qdata/dd/nodekey -writeaddress`

        # Add the enode to static-nodes.json
        sep=`[[ $ip != ${ips[-1]} ]] && echo ","`
        echo '  "enode://'$enode'@'$ip':30303?discport=0"'$sep >> static-nodes.json

        let n++
    done
    echo "]" >> static-nodes.json

## Create Ethereum accounts and *genesis.json* file

To allow nodes to send transactions they will need some Ether. This is required in Quorum, even though gas is zero cost. For simplicity we create an account and private key for each node, and we create the genesis block such that each of the accounts is pre-cherged with a billion Ether (10^27 Wei). 
**Everything after initial balances is consensus-mechanism-specific, so we exchanged it for an Istanbul-Consensus-compatible part.**

The Geth executable in the Docker image is used to create the accounts. An empty *passwords.txt* file is created which is used when unlocking the (passwordless) Ether account for each node when starting Geth in *start-node.sh*.

    #### Create accounts, keys and genesis.json file #######################

    cat > genesis.json <<EOF
    {
      "alloc": {
    EOF

    n=1
    for ip in ${ips[*]}
    do
        qd=qdata_$n

        # Generate an Ether account for the node
        touch $qd/passwords.txt
        account=`docker run -u $uid:$gid -v $pwd/$qd:/qdata $image /usr/local/bin/geth --datadir=/qdata/dd --password /qdata/passwords.txt account new | cut -c 11-50`

        # Add the account to the genesis block so it has some Ether at start-up
        sep=`[[ $ip != ${ips[-1]} ]] && echo ","`
        cat >> genesis.json <<EOF
        "${account}": {
          "balance": "1000000000000000000000000000"
        }${sep}
    EOF

        let n++
    done

    cat >> genesis.json <<EOF
      },
    "coinbase": "0x0000000000000000000000000000000000000000",
    "config": {
        "homesteadBlock": 1,
        "eip150Block": 2,
        "eip150Hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
        "eip155Block": 3,
        "eip158Block": 3,
        "istanbul": {
        "epoch": 30000,
        "policy": 0
        },
        "isQuorum": true
    },
    "extraData": "[PASTE YOUR ISTANBUL-TOOLS GENERATED EXTRADATA (SPECIFIC FOR YOUR NODE-KEYS) HERE]",
    "gasLimit": "0x47b760",
    "difficulty": "0x1",
    "mixHash": "0x63746963616c2062797a616e74696e65206661756c7420746f6c6572616e6365",
    "nonce": "0x0",
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "timestamp": "0x00"
    }   
    EOF

** It should be noted here that, once the setup script has run, you will have access to the keys generated for each Node. You will need to create a config.toml file which contains the following information:

    vanity = "0x0000000000000000000000000000000000000000000000000000000000000000"
    validators = ["0xNode1Address",
    "0xNode2Address",
    "0xNode3Address",
    ...]

You will then need to run the command...
    istanbul extra encode --config ./config.toml
... made available to you here: https://github.com/getamis/istanbul-tools

This is important since the extraData field in your genesis-file encodes the Addresses of the Nodes that are validators in your Istanbul network.

You can find the addresses for your node in the genesis-file for your network - just pick any of them and the nodes will be listed there since you are pre-funding them there. Just ensure that they are hex (add a 0x in front of the address if it is not there yet).

It is important that you complete this step BEFORE you docker-compose up -d the first time! **

The account created for each node will be available as `eth.accounts[0]` in the node's console.

## List node IP addresses for the Quorum transaction manager, *tm.conf*

The Quorum transaction manager currently needs to know the IP addresses of peers it may need to send private transactions to. We list them out here. Each node will have the same list - it ignores its own IP address. The transaction manager process is hosted on port 9000.

    #### Make node list for tm.conf ########################################

    nodelist=
    n=1
    for ip in ${ips[*]}
    do
        sep=`[[ $ip != ${ips[0]} ]] && echo ","`
        nodelist=${nodelist}${sep}'"http://'${ip}':9000/"'
        let n++
    done

## Further configuration

    #### Complete each node's configuration ################################

    n=1
    for ip in ${ips[*]}
    do
        qd=qdata_$n

*tm.conf* is the transaction manager configuration file for each node. We use a pre-populated template for this, inserting the IP address of the node and the list of peer nodes created above.

        cat templates/tm.conf \
            | sed s/_NODEIP_/${ips[$((n-1))]}/g \
            | sed s%_NODELIST_%$nodelist%g \
                  > $qd/tm.conf

We copy into each node's directory the *genesis.json* and *static-nodes.json* files that were created earlier.

        cp genesis.json $qd/genesis.json
        cp static-nodes.json $qd/dd/static-nodes.json

Quorum's Constellation needs public/private keypairs to operate. The *tm.pub* key is the address to which "privateFor" transactions should be sent for a node. Quorum provides a utility for generating these keys, and again we use the instance in the Docker image. I believe the *tma.{pub,key}* files are being deprecated, but they are still needed for the time-being.

        # Generate Quorum-related keys (used by Constellation)
        docker run -u $uid:$gid -v $pwd/$qd:/qdata $image /usr/local/bin/constellation-enclave-keygen /qdata/keys/tm /qdata/keys/tma < /dev/null > /dev/null
        echo 'Node '$n' public key: '`cat $qd/keys/tm.pub`

        cp templates/start-node.sh $qd/start-node.sh
        chmod 755 $qd/start-node.sh

        let n++
    done
    rm -rf genesis.json static-nodes.json

## Create *docker-compose.yml*

#### Create the docker-compose file ####################################

This is the first file that is not written to the node-specific directories. This will be used by *docker-compose* to start and stop the containers and network. Each node/container has an entry.

    cat > docker-compose.yml <<EOF
    version: '2'
    services:
    EOF

    n=1
    for ip in ${ips[*]}
    do
        qd=qdata_$n

        cat >> docker-compose.yml <<EOF
      node_$n:
        image: $image
        volumes:
          - './$qd:/qdata'
        networks:
          quorum_net:
            ipv4_address: '$ip'
        ports:
          - $((n+22000)):8545
        user: '$uid:$gid'
    EOF

        let n++
    done

    cat >> docker-compose.yml <<EOF

    networks:
      quorum_net:
        driver: bridge
        ipam:
          driver: default
          config:
          - subnet: $subnet
    EOF
