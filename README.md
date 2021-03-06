- [Intro](#fluence-demo-december-2018)
- [Prerequisites](#prerequisites)
- [Scenario](#scenario)
    - [Clone repo](#clone-repo)
    - [Run Ethereum blockchain](#run-ethereum-blockchain)
    - [Run Swarm & 4 Fluence nodes](#run-swarm--4-fluence-nodes)
    - [Publish WASM code to Ethereum contract](#publish-wasm-code-to-ethereum-contract)
    - [Formation of the real-time cluster](#formation-of-the-real-time-cluster)
        - [Emitting the event](#emitting-the-event)
    - [Try web application](#try-web-application)
        
[![Join the chat at https://gitter.im/fluencelabs/workshop](https://badges.gitter.im/fluencelabs/workshop.svg)](https://gitter.im/fluencelabs/workshop?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

## Fluence Demo: December 2018
This demo will allow you to play with Fluence Nodes and Real-Time Clusters. You will run Fluence from scratch, manage it through Ethereum smart contract, create real-time clusters with decentralized database written in Rust, and use it through HTTP API with a simple web application.

## Prerequisites
You will need installed `Docker` and `npm`.

**NOTE:** this _`docker-compose.yml` and `fluence-cli` work only on macOS. If you would like to run it on other OS, you will need to change `host.docker.internal` to respective IP addresses in `docker-compose.yml` and [build CLI for you OS](https://github.com/fluencelabs/fluence/tree/master/cli)._

## Scenario
### Clone repo
First of all, clone this repo
```
git clone https://github.com/fluencelabs/demo-2018-dec
```

### Run Ethereum blockchain
Then, go to `bootstrap` directory and run Ganache blockchain:
```
cd bootstrap
npm install
pkill -f ganache # kill Ganache if it was running already
npm run ganache > /dev/null
npm run migrate
```

Note the contract address in the output, you will need it later:
> Deployer: 0x9995882876ae612bfd829498ccd73dd962ec950a

Another piece of hex you will need is current Ethereum account in Ganache:
```
# while in bootstrap directory
npm run getEthAccount
```

### Run Swarm & 4 Fluence nodes
Now it's time to run Swarm and 4 Fluence nodes! That's easy:
```
cd .. # go back to repository root directory
docker-compose up -dV
```

You can check out running containers by running `docker ps` in terminal. Also, if logs are of interest for you, run `docker logs node1`. 

On start, nodes will register themselves in Ethereum contract at address `0x9995882876ae612bfd829498ccd73dd962ec950a`, signaling ready to get some work. See image below.
<div style="text-align:center">
<br>
<img src="img/master_node_registration.png" width="764px"/>
<br><br><br>
</div>

### Publish WASM code to Ethereum contract
To give these nodes work, we'll take [llamadb](https://github.com/nukep/llamadb/), a database written in Rust, compiled to WebAssembly, and upload it to Swarm. Resulting address should be sent to Ethereum contract along with the size of a cluster we wish to be serving our code. 

<div style="text-align:center">
<br>
<img src="img/cli_swarm_contract.png" width="665px"/>
<br><br><br>
</div>

We'll use [Fluence CLI](https://github.com/fluencelabs/fluence/tree/master/cli) for that:
```
# while in repository root directory
./fluence-cli publish llama_db.wasm 0x9995882876ae612bfd829498ccd73dd962ec950a 0x4180fc65d613ba7e1a385181a219f1dbfe7bf11d --cluster_size 4
```

You may take a look at `./fluence-cli publish --help` to get the idea of how to use it.

### Formation of the real-time cluster
After we published code, Ethereum contract will match the code to registered nodes and emit an event singaling them to form real-time cluster.

<div style="text-align:center">
<br>
<img src="img/contract_match.png" width="788px"/>
<br><br><br>
</div>

#### Emitting the event
Event contains addresses and ports of all four future cluster members. On receiving an event, nodes will run real-time nodes in docker containers, specifying peers in the configuration files. Note that all containers are running on the host, there is no Docker-in-Docker here.

<div style="text-align:center">
<br>
<img src="img/cluster_creation.png" width="766px"/>
<br><br><br>
</div>

You may take a look at real-time node's logs `docker logs 01_node2`. Look for `height=2`, that means that Tendermint produced 2 initial blocks and cluster is ready to receive transactions.

### Try web application
Now let's run a simple web application which will connect to real-time cluster through [Javascript Fluence library](https://github.com/fluencelabs/fluence/tree/master/js-client). Go to `sql-client` and open `index.html` in a web browser.

You will see real-time nodes' statuses on the left:
<div style="text-align:left">
<br>
<img src="img/web_status.png" width="357px"/>
<br><br><br>
</div>

Copy all queries from the `Example queries` block to the `Type queries` block:
  
<img src="img/tips_block.png" width="600"/>

Push the `Submit query` button:
  
<img src="img/queries_block.png" width="600"/>

Wait for the result in the `Result` block:
  
<img src="img/results_block.png" width="600"/>

You can see logs of what's happening in the web console:
  
<img src="img/console_block.png" width="1065"/>

### Cleaning the data
To clean containers and related volumes, run
```
docker ps -a | awk '{print $1}' | xargs docker rm -f ; docker volume prune -f
```

To remove all `docker-compose` related data, run
```
docker-compose kill
```

To stop Ganache, run
```
pkill -f ganache
```

And that's it.