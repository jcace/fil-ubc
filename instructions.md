# UBC Decentralized Storage Networks - Practicum

## Introduction
In this session, we'll walk through using IPFS and Filecoin to demonstrate how the network works.

[IPFS Tutorial](https://curriculum.pl-launchpad.io/curriculum/ipfs/content-addressing/)

Before beginning, Join the Filecoin slack! We'll use it to communicate and it's a great place to follow up and ask questions later. 
[Filecoin Slack](https://filecoinproject.slack.com/)

# IPFS

## DAG & CID Exploration

### Examining the DAG
Let's go to `dag.ipfs.io`, and upload a file here

Upload a file, and you can see it generate the DAG.
Play around with the chunks, as well as the # of children and see how the CIDs / nodes change. You'll see a visual representation of the Merkle DAG that's constructed from the file.

> Hint: Use a large text file of a few kB max to see a nice DAG. Larger files, such as photos, will take an extremely long time to render.

### Inspecting the CIDs 
Click on a CID, and see the human readable breakdown of its format and the self-describing headers.

## Hands-on with IPFS
First, install the reference implementation of IPFS, Kubo.

First download from here
[Download Links](https://dist.ipfs.tech/#kubo)

Then, 
```bash
ipfs init
```

This will set up private keys so you can run your own IPFS node. 

Run the daemon
```bash
ipfs daemon
```

Leave it running in the background, so we can interact with it.

### Pinning a file

Make a text file
```text
Hello {Your name!}
```

Add it to IPFS
```bash
ipfs add hello.txt
```

Note the CID that comes back:

```bash
added QmaKXZPnQFnL1L42WKhfMiWYoRGEGjvB7pKF3KqxRDk599 hello.txt
```

Now, we can query it back:

```bash
ipfs cat QmaKXZPnQFnL1L42WKhfMiWYoRGEGjvB7pKF3KqxRDk599
```

Try from a friend's computer - it should work and you can can see the same file! 

#### Troubleshooting - It doesnt work
If you can't seem to query the file back, it's likely because there file can't be found in the DHT. This is because there hasn't been enough time for the advertisements to propagate between you and your peer. We could either wait, and hope that our content gets propagates through the network, or we can simply add a direct peer so the connection is made immediately.

**On the IPFS peer where data is pinned**, find your peer ID:
```bash
ipfs id
```

Then, Grab the first entry form the `Addresses` array. It should look like this:
```bash
/ip4/10.10.10.10/tcp/4001/p2p/12D3.....
```

Now, on the IPFS peer where you want to query the data from, run:
```bash
ipfs swarm connect /ip4/10.10.10.10/tcp/4001/p2p/12D3
```

Re-try the `ipfs cat`, and you should be able to download the file now!

### Garbage collect the CID
`ipfs repo gc` will force a garbage-collection, and remove the file from the repository. Do this from the first node, but leave the first node up. Try to query from a third node, and the content should still be accessible!

> Hint: In practice, if you want to avoid a piece of content from being garbage collected, you can run `ipfs pin add QmaKXZPnQFnL1L42WKhfMiWYoRGEGjvB7pKF3KqxRDk599` for example

# Filecoin
Making a storage deal on Filecoin requires a couple steps. 




## Source data

For simplicity, we will generate some random data for testing purposes. 
```bash
dd if=/dev/random of=testfile bs=500000000 count=2
```

This will generate a file  1GB in size (1 GiB)

You can totally use your own filesystem if you want - an entire directory tree can be prepared if you want!

## Data Preparation
First, we need to prepare the data into a format called [CAR](https://ipld.io/specs/transport/car/carv1/) (Content Addressible Archive), which is a **serialization of the DAG to be stored**

There are a few ways to do this depending on the exact requirements, as a multitude of tools exist in the ecosystem. Here are some examples:
https://github.com/anjor/go-fil-dataprep
https://github.com/application-research/delta
https://github.com/data-preservation-programs/singularity
https://github.com/tech-greedy/generate-car
https://github.com/tech-greedy/js-generate-car
https://github.com/web3-storage/ipfs-car
https://github.com/schreck23/ptolemy

[Carfile dataprep explanation](https://github.com/filecoin-project/data-prep-tools/tree/main/docs)

For our purposes, we'll use `go-fil-dataprep` as it's easy to install and lightweight to run. 


First, Install go-fil-dataprep

`go install github.com/anjor/go-fil-dataprep/cmd/data-prep@latest`


Next, genereate the carfile
`data-prep fil-data-prep --output test --size 4026531840 testfile`

> note, you can also provide a directory as the root to generate a carfile of the entire directory tree, for example /path/to/dir

The Size flag is the max size of individual carfiles. The number provided is 30GiB. As the maximum deal size on Filecoin is 32GiB, this leaves plenty of room for overhead and is a safe value to use.

This will produce a car file named after the `CID.car` of the file.

ex - `test-baga6ea....car`
Take a note of the `root cid` that's output to the terminal. This is the root CID of the data (also known as the Payload CID). The filename contains another CID (Piece CID), which is the CID of the carfiles (it also includes some headers)


## Making a Filecoin deal
Now, we can make a deal with the Filecoin network.

### Prerequisites
We need a couple pieces of software installed first for the tools we'll be using.

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
wget -c https://golang.org/dl/go1.19.7.linux-amd64.tar.gz -O - | sudo tar -xz -C /usr/local
```


### Install Boost Client
We'll use Boost client
[Boost Client](https://boost.filecoin.io/getting-started/boost-client), 
Boost is an approachable way of making deals with the Filecoin network.

You need `Go` and `Rust` installed first (see above).

Install it - you have to build from source

```bash
# Remove any existing boost implementations
rm -rf ~/.boost
git clone https://github.com/filecoin-project/boost.git
cd boost
make build
```

Make sure it works:
```bash
./boost
```

In order to use Boost, you have to point it to a Full Filecoin Chain RPC Node. The easiest way to do this is to point it at a public API. To do this, set the following environment variable (Glif is a reliable public node that's available for free)

```bash
export FULLNODE_API_INFO=https://api.node.glif.io
```


### Initialize Boost
Boost has a local database/folder that you need to initialize first. This will create the local repo in `~/.boost`. 

```bash
./boost init
```

This command will also create a Filecoin wallet - the public address of the wallet will be displayed to you. For example:

```bash
default wallet set	{"wallet": "f3q2ac6l6wih7e3bk25mdb7stqnhvw35gfcccdn7bd65fnhebaxgvxq7av3dpjnkikrebyzkhzijdddeq7gbja"}
```

Save this wallet address somewhere, we'll need it later.

### Getting Datacap
*Datacap* is like a Filecoin Token that gives your deals Fil+ (Verified) status. This allows you to make deals without any cost, as providers gain increased ROI for taking them.
You can get a small amount of Datacap for free from the faucet. Do this here: 

[Glif Faucet](https://verify.glif.io/)

You'll have to sign in with your Github account for to get the Datacap.
Paste in your wallet and click "Request"

Wait a few minutes, and you should see the datacap if you do the Status Check.

You can double check that Boost sees the datacap too, by running:

```bash
./boost wallet list
```

### Finding a storage provider
You have to know which SP you want to make a deal with. There are a couple of public indexes available. All storage providers are identified with an id in the form `f0XXXXXXX`. 

One such index is at https://data.storage.market - click Providers/all 
Another one is https://filrep.io

Eventually, this process will be automatic via [Spade](https://github.com/data-preservation-programs/spade)

For purposes of this test, we can use a known storage provider - `f01963614`

### Hosting your car file
In order to make the deal, we need to provide a way for the storage provider to download the carfile from us. Commonly, this is done over regular old HTTP or FTP.
> **Note** There are ways of doing with via IPFS, but for purposes of this demo, it's easier to simply transfer over HTTP. See `Delta Approach` below for more info on the IPFS approach.

Running a file server is trivial with software like Nginx, Caddy, or even a simple Nodejs/Web framework. For purposes of this demo, however (to avoid the pitfalls of trying to forward traffic through a University student firewall :) ), we'll just use a pre-computed one that I've uploaded to a local webserver. It's located at:

`https://10.32.32.20:2023/baga6ea4seaqpu5vhfki6isgsbukjmtuap4waga57k7n4v57fao4oq3v2k2koqdq.car`

The metadata for this carfile is as follows:
- Piece CID: `baga6ea4seaqpu5vhfki6isgsbukjmtuap4waga57k7n4v57fao4oq3v2k2koqdq`
- Payload (root) CID: `bafybeiecfinwq6zoy7hrwhjvnqiba3k4an452z6hrz37p7rnlvamlpvhxe`
- Size: 1000085081 bytes


### Making the deal!
Now, we can make the Filecoin deal. 

```bash
./boost deal --provider f01963614 --car-size 1000085081 --piece-size 1073741824 --http-url http://10.32.32.20:2023/baga6ea4seaqpu5vhfki6isgsbukjmtuap4waga57k7n4v57fao4oq3v2k2koqdq.car --payload-cid bafybeiecfinwq6zoy7hrwhjvnqiba3k4an452z6hrz37p7rnlvamlpvhxe --commp baga6ea4seaqpu5vhfki6isgsbukjmtuap4waga57k7n4v57fao4oq3v2k2koqdq --storage-price 0 --verified --remove-unsealed-copy
```

Breaking down the flags of this request:
- `provider` is the ID of the storage provider we want to make the deal with, determined earlier
- `car-size` is taken from the raw file size (as seen on the filesystem, or https://fwscdn.estuary.tech/delta-test/)
- `piece-size` is the *next larger power of two*, greater than the car size. This is a low-level detail of how Filecoin packs pieces into sectors, but they must all align to power-of-two sizes so that everything fits cleanly.
- `http-url` is the URL where to download the CAR from. It must be reachable by the Storage Provider. (Note: This can also be secured and auth credentials may be passed in via `http-headers` flag)
- `payload-cid` is the CID of the content that's being stored. This is the Root CID the DAG, and would map to the IPFS root CID.
- `commp` is the CID of the CAR file. Think of the carfile as a "container" for the data, including some additional headers. This is the CID of the container.
- `storage-price` is the price in FIL per GiB per Epoch. Since we'll use Datacap (Fil+) for this deal, we set it to 0.
- `verified` is a flag that indicates that we're using Datacap (Fil+) for this deal.
- `remove-unsealed-copy` is a flag that indicates that we don't want to keep *duplicate* copy alongside the sealed data. When this flag is unspecified, the provider can serve back the file immediately when requested, at the cost of additional storage space.


See the result:

```bash
sent deal proposal
  deal uuid: 4edc998b-ba87-45c9-9162-4d566212ef13
  storage provider: f01963614
  client wallet: f3q2ac6l6wih7e3bk25mdb7stqnhvw35gfcccdn7bd65fnhebaxgvxq7av3dpjnkikrebyzkhzijdddeq7gbja
  payload cid: bafybeiecfinwq6zoy7hrwhjvnqiba3k4an452z6hrz37p7rnlvamlpvhxe
  url: http://10.32.32.20:2023/baga6ea4seaqpu5vhfki6isgsbukjmtuap4waga57k7n4v57fao4oq3v2k2koqdq.car
  commp: baga6ea4seaqpu5vhfki6isgsbukjmtuap4waga57k7n4v57fao4oq3v2k2koqdq
  start epoch: 3041396
  end epoch: 3559796
  provider collateral: 320.331 μFIL
```

The deal has been sent off to the provider, and it will soon be sealed and stored on the network!

# Retrieving Data
https://github.com/filecoin-project/lassie/


## Install and build
Download the correct one for your OS from here: [Lassie downloads page](https://github.com/filecoin-project/lassie/releases)

Then, you can easily use it to fetch from Filecoin:

```bash
lassie fetch bafybeiecfinwq6zoy7hrwhjvnqiba3k4an452z6hrz37p7rnlvamlpvhxe
```


# Delta Approach

Download and install

Set up a .env file


```bash
./delta daemon --mode standalone
```


Then make the deal,
```bash
curl --location --request POST 'http://localhost:1414/api/v1/deal/end-to-end' \
--header 'Authorization: Bearer ESTc978971e-05a3-475b-ba13-7081317ddef6ARY' \
--form 'data=@"/Users/jc/Dev/TEMP/car-gen/testfile"' \
--form 'metadata="{\"miner\": \"f01963614\",\"connection_mode\": \"e2e\",\"wallet\": {\"address\":\"f3q2ac6l6wih7e3bk25mdb7stqnhvw35gfcccdn7bd65fnhebaxgvxq7av3dpjnkikrebyzkhzijdddeq7gbja\"}}"'
```