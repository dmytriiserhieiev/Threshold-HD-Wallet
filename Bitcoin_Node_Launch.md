Steps for launching a Bitcoin Full Node on AWS EC2. Adapted from https://wolfmcnally.com/115/developer-notes-setting-up-a-bitcoin-node-on-aws/, 
and updated to reflect what worked best to launch my Bitcoin Full Node.  

### Initial Steps

* From the AWS console, navigate to EC2.
* Click the Launch Instance button.

## Step 1: Choose an Amazon Machine Image (AMI)

* Click on the AWS Marketplace tab.
* Search for `Debian Stretch` in the search box.
* Find the AMI for *Debian GNU/Linux 9 (Stretch)* and click **Select**. 
* A box will appear with details and a list of example prices. Click **Continue**.

## Step 2: Names and Tags

* In the *Name* field, enter something like `Bitcoin Node`. 

## Step 3: Choose an Instance Type

You will need an instance with 2-3 gigabytes of memory. 

* Choose the `t3.medium` instance, which has 4GB of memory and can be connected to EBS (Elastic Block Storage) of any size. 

## Step 4: Create Key-Pair Login

* Click on **Create New Key Pair**. Enter the key pair name, choose the key pair type, the private key file format, and click **Create key pair**.
* The key pair file downloads on your computer.

## Step 5: Network Settings & Security Group

* Select Create a New Security Group.
* Enter a Name and Description for the Security Group. For example, *Bitcoin Node* and *Ports and services necessary for running a Bitcoin Node*. 
* A rule allowing SSH access to your instance from any other IP address is already in place. The bitcoin client needs to talk through port 8333 for Mainnet and 18333 for Testnet. We will create rules for allowin them both, as well as a rule that opens the port used by the Lightning Protocol. 
* Click the **Add Rule** button. 
* Select **Custom UDP** for the *Type*. 
* Enter `8333` for *Port Range*.
* Select **Anywhere** for *Source*. 
* Enter `Mainnet` in the *Description* field. 
* Perform the last five steps again, but use `18333` for the *Port Range* and `Testnet` for the *Description*. 
* Do it one more time, but use `9735` for the *Port Range* and `Lightning` for the *Description*. 

## Step 6: Configure Storage

* Click **Add New Volume**.
* Choose `EBS`, `/dev/sdb` for the *Device* column, 300 GB for the *Size*, and **General Purpose SSD** for the *Volume Type*. 
* Launch the Instance.

## Launching

The next page keeps you informed on the launch status. Assuming all goes well, you will receive the message Your instances are now launching and instance ID like this: `i-0d881a693cb29c072`.

* Click **View Instances**.

You should see your new `Bitcoin` instance in the list. Eventually the *Instance State* will change to `Running`. For awhile the *Status Checks* column will also say `Initializing`. 
Wait for this to change (reload the page if necessary) to `2/2 checks passed`.

## Log In to Your Instance for the First Time

From the View Instances page, click on your new instance in the table of instances. At the top you will find a **Public IPv4 address** field with an IP address like: xx.xx.xx.xx. 
This is an ephemeral IP address that will change if you stop and restart your instance. You can set up an EC2 Elastic IP address to keep this from happening (not covered here.)

Prior to SSH'ing into your instance, you will need to modify the pem key file to have certain secure properties. This ensures no other users can easily access the instance. To do this,
simply run the following command for your file:

```
$ sudo chmod 600 /path/to/my/key.pem
```

If you try to SSH into the instance without modifying the key, you will incur an **Unprotected Private Key File Warning**. To copy the file path in Mac, open the file location in a Finder Window. 
Hold down the **Control** button, and left click on the pem key file. While the **Control** button is held down, simultaneously press the **Option** button to show
the menu option *Copy "file" as Pathname*. Select this option, then you can paste into the terminal. 

Note: I had to move my pem file onto my Desktop before the Mac allowed me to run `chmod 600` on it. 

In the terminal:

```
$ ssh -i /path/to/your/keypair.pem admin@xx.xx.xx.xx
```

For EC2 VM, use `ssh -i /path/to/your/keypair.pem ec2-user@xx.xxx.xxx.xx` instead. Otherwise, you will incur the following error: `Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`

Successful login should show something like:

```
Linux ip-172-31-56-252 4.9.0-14-amd64 #1 SMP Debian 4.9.240-2 (2020-10-30) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Feb  2 15:03:11 2021 from 195.181.167.196
admin@ip-172-31-56-252:~$ 
```

## Installing Updates

The first thing to do is update everything that needs updating since this distro was produced. To do so, you must first enter the root account:

```
$ sudo -s
$ sudo apt-get update && apt-get upgrade
```

Set up future updates to happen automatically:

```
$ sudo echo "unattended-upgrades unattended-upgrades/enable_auto_updates boolean true" | debconf-set-selections
$ sudo apt-get -y install unattended-upgrades
```

Install a Random Number Generator

```
$ sudo apt-get install haveged -y
```

## Install Bitcoin

Ensure you are working as `admin` and not `root`. To exit root shell, type `Ctrl-D`.

### Add useful aliases

Edit `.bash_profile` to add useful aliases. Run `ls -la` to check the contents of your instance for the `.bash_profile` file. If it does not exist, create the file. 

```
touch .bash_profile
```

Run `vim .bash_profile` to edit the file for aliases. After editing the file, press **ESC**. Then, while holding down **Shift**, press `:`. Type `wq` and press enter. 

```
alias btcdir="cd ~/.bitcoin/" #linux default bitcoind path
alias bc="bitcoin-cli"
alias bd="bitcoind"
alias btcinfo='bitcoin-cli getwalletinfo | egrep "\"balance\""; bitcoin-cli getinfo | egrep "\"version\"|connections"; bitcoin-cli getmininginfo | egrep "\"blocks\"|errors"'
alias btcblock="echo \\\`bitcoin-cli getblockcount 2>&1\\\`/\\\`wget -O - http://blockexplorer.com/testnet/q/getblockcount 2> /dev/null | cut -d : -f2 | rev | cut -c 2- | rev\\\`"
```

### Set up some some shell variables

Set up two variables to make this installation more automatic. The first variable, `$BITCOIN`, should be set to the current version of Bitcoin. The second will then automatically generate a truncated form used by some of the files. 

```
$ export BITCOIN=bitcoin-core-0.21.1
$ export BITCOINPLAIN=`echo $BITCOIN | sed 's/bitcoin-core/bitcoin/'`
```

### Download Files

```
$ wget https://bitcoin.org/bin/$BITCOIN/$BITCOINPLAIN-x86_64-linux-gnu.tar.gz
$ wget https://bitcoin.org/bin/$BITCOIN/SHA256SUMS.asc
$ wget https://bitcoin.org/laanwj-releases.asc
```

### Verify Bitcoin Signature

```
$ /usr/bin/gpg --import laanwj-releases.asc
$ /usr/bin/gpg --verify SHA256SUMS.asc
```

Amongst the information you get back from the last command, should be a line telling you that you have a "Good Signature". Do not worry about the warning. 

### Verify Bitcoin SHA

Next, you should verify the Hash for the Bitcoin tar file against the expected Hash:

```
$ /usr/bin/sha256sum $BITCOINPLAIN-x86_64-linux-gnu.tar.gz | awk '{print $1}'
$ cat SHA256SUMS.asc | grep $BITCOINPLAIN-x86_64-linux-gnu.tar.gz | awk '{print $1}'
```
If those both produce the same number, it is OK. 

### Install Bitcoin

```
$ /bin/tar xzf $BITCOINPLAIN-x86_64-linux-gnu.tar.gz
$ sudo /usr/bin/install -m 0755 -o root -g root -t /usr/local/bin $BITCOINPLAIN/bin/*
$ /bin/rm -rf $BITCOINPLAIN/
```

### Create Bitcoin Configuration File

Create a Bitcoin configuration file for the node. Whenever the user updates and saves the configuration file, the Bitcoin node must be restarted in order for the new configuration settings to take effect. 

```
$ mkdir .bitcoin
$ touch .bitcoin/bitcoin.conf
$ vim bitcoin.conf
```

Paste the following settings into your `bitcoin.conf` file:

```
# Accept command line and JSON-RPC commands
server=1
{1}
# Set database cache size in megabytes (4 to 16384, default: 450)
dbcache=1536
{1}
# Set the number of script verification threads (-6 to 16, 0 = auto, <0 = leave that many cores free, default: 0)
par=1
{1}
# Set to blocksonly mode, sends and receives no lose transactions, instead handles only complete blocks
blocksonly=1
{1}
# Tries to keep outbound traffic under the given target (in MiB per 24h), 0 = no limit (default: 0)
maxuploadtarget=137
{1}
# Maintain at most <n> connections to peers (default: 125)
maxconnections=16
{1}
# Username for JSON-RPC connections
rpcuser=bitcoinrpc
{1}
# Password for JSON-RPC connections
rpcpassword=$(xxd -l 16 -p /dev/urandom)
{1}
# Allow JSON-RPC connections from, by default only localhost are allowed
rpcallowip=127.0.0.1
{1}
# Use the test chain
testnet=1
{1}
# Maintain a full transaction index, used by the getrawtransaction rpc call (default: 0)
txindex=1
{1}
# Make the wallet broadcast transactions (default: 1)
walletbroadcast=1
EOF
```

**Testnet vs Mainnet**: If you want to use mainnet instead of testnet, just omit the `testnet=1` line. 

Limit permissions to your configuration file:

```
$ /bin/chmod 600 .bitcoin/bitcoin.conf
```

### Start the Bitcoin Daemon

```
$ bitcoind -daemon
```



