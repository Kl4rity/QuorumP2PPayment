# quorum-docker-Nnodes-Istanbul

DISCLAIMER: This is an existing example (created by ConsenSys) adjusted for the Istanbul consensus mechanism. Please find the original repo here: https://github.com/ConsenSys/quorum-docker-Nnodes

Run a bunch of Quorum nodes using Istanbul, each in a separate Docker container.

This is simply a learning exercise for configuring Quorum networks. Probably best not used in a production environment.

## Building

In the top level directory:

    docker build -t quorum .
    
The first time will take a while, but after some caching it gets much quicker for any minor updates.

## Running

Change to the *Nnodes/* directory. Edit the `ips` variable in *setup.sh* to list two or more IP addresses on the Docker network that will host nodes:

    ips=("172.13.0.2" "172.13.0.3" "172.13.0.4")

The IP addresses are needed for Constellation to work. Now run,

    ./setup.sh

Create extraData from key-information with Istanbul-Tools - please refer to readme in Nnodes Folder.

    docker-compose up -d
    
This will set up as many Quorum nodes as IP addresses you supplied, each in a separate container, on a Docker network, all hopefully talking to each other.

    Nnodes> docker ps
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                     NAMES
    83ad1de7eea6        quorum              "/qdata/start-node.sh"   55 seconds ago      Up 53 seconds       0.0.0.0:22002->8545/tcp   nnodes_node_2_1
    14b903ca465c        quorum              "/qdata/start-node.sh"   55 seconds ago      Up 54 seconds       0.0.0.0:22003->8545/tcp   nnodes_node_3_1
    d60bcf0b8a4f        quorum              "/qdata/start-node.sh"   55 seconds ago      Up 54 seconds       0.0.0.0:22001->8545/tcp   nnodes_node_1_1

## Stopping

    docker-compose down
  
## Playing

### Accessing the Geth console

If you have Geth installed on the host machine you can do the following from the *Nnodes* directory to attach to Node 1's console.

    geth attach qdata_1/dd/geth.ipc

Otherwise, the following will achieve the same thing, attaching via the Geth instance in the container.  If you do this, you'll have to copy transaction scripts used below into the *qdata_N* directories manually.

    docker exec -it nnodes_node_1_1 geth attach /qdata/dd/geth.ipc
