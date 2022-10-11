# Setup cardano-node on Debian/Ubuntu as a relay
This document assumes that you have already installed a minimal Debian/Ubuntu system, preferably without any windowing system or other unnecessary software.

During the standard Debian install at the software selection stage, you only need:
* 'base system'
* 'ssh server'

Ensure you know how to use ssh to log into your relay machine and scp to copy files to it, preferably using key based authentication.

## 1. Build cardano-node deb package
From a security perspective, it is better to keep the software installed on your relay to the bare minimum required.  Therefore it is better to build your deb packages on another machine so that the compiler and build chain is not required on your relay.

Follow instructions for building the cardano-node deb package:
* https://github.com/TerminadaPool/libsecp256k1-iog-debian
* https://github.com/TerminadaPool/cardano-node-debian

Copy the cardano-node deb package to your minimal Debian/Ubuntu system that will hereby be referred to as 'relay1'.

## 2. Install required packages
```
apt install libsodium23 libnuma1 curl jq chrony
```

You may also want the following extra packages:
```
apt install rsync tmux
```

## 3. Install cardano-node deb
```
dpkg -i cardano-node_1.35.3-1_arm64.deb
```

## 4(a). Configure cardano-node to run in P2P mode
Peer to peer mode works really well currently, and eventually all Cardano nodes will connect to each other using the P2P method.  However, P2P mode is not yet officially recommended as the default mode for stake pool operators to run.  Stake pool operators can still run a relay in P2P mode to help with testing, but they should run other relays in non-P2P mode.

Edit /etc/cardano/mainnet-config.json and add the following after "ShelleyGenesisHash" line:  
```
  "TestEnableDevelopmentNetworkProtocols": true,
  "EnableP2P": true,
  "MaxConcurrencyBulkSync": 2,
  "MaxConcurrencyDeadline": 2,
  "TargetNumberOfRootPeers": 100,
  "TargetNumberOfKnownPeers": 100,
  "TargetNumberOfEstablishedPeers": 50,
  "TargetNumberOfActivePeers": 20,
```
Then change these other settings
```
  "TraceBlockFetchDecisions": true,
  "TraceMempool": false,
  "hasPrometheus": [
    "0.0.0.0",
    12798
  ],
```

Edit /etc/cardano/mainnet-topology.json to look like:  
```
{
  "LocalRoots": {
    "groups": [
      {
        "localRoots": {
          "accessPoints": [],
          "advertise": false
        },
        "valency": 0
      }
    ]
  },
  "PublicRoots": [
    {
      "publicRoots" : {
        "accessPoints": [
          {
            "address": "relays-new.cardano-mainnet.iohk.io",
            "port": 3001
          }
        ],
        "advertise": true
      },
      "valency": 1
    }
  ],
  "useLedgerAfterSlot": 0
}
```

## 4(b). Configure cardano-node to run in non-P2P mode
Choose this option if you don't want to run your relay in P2P mode.

Edit /etc/cardano/mainnet-config.json and change the following settings:  
```
  "TraceBlockFetchDecisions": true,
  "TraceMempool": false,
  "hasPrometheus": [
    "0.0.0.0",
    12798
  ],
```

Edit /etc/cardano/mainnet-topology.json to look like:  
(Note: You shouldn't have to change anything from the default downloaded version)  
```
{
  "Producers": [
    {
      "addr": "relays-new.cardano-mainnet.iohk.io",
      "port": 3001,
      "valency": 2
    }
  ]
}
```

## 5. Start cardano-node
First check you are happy with the default systemd service file at: /lib/systemd/system/cardano-node.service

Then enable and start cardano-node
```
systemctl enable cardano-node
systemctl start cardano-node
```

## 6. Check status cardano-node synchronisation
```
CARDANO_NODE_SOCKET_PATH='/run/cardano/mainnet-node.socket' cardano-cli query tip --mainnet
```
When fully synchronised with the blockchain the output will look like:  
> {
>     "block": 7857531,
>     "epoch": 368,
>     "era": "Babbage",
>     "hash": "a2c897d15bf31644c5797814c0fefc29bfa5c7f4e3d1747fcb4c1789dc330d52",
>     "slot": 73670460,
>     "syncProgress": "100.00"
> }

## DONE
If you only need a running cardano-node synchronised to the blockchain then you are now done.

****
****

But if you need this relay to be part of a stake pool operation, or you are running your node in non-P2P mode, then you will need to update your topology configuration from time to time.  Continue on below:

## 7. Regularly ping api.clio.one with your relay's connection information
Wait for your relay to be fully synchronised with the current blockchain before doing this step.

This step is not essential if you are running in P2P mode.  However, it is recommended while the majority of relays on the cardano network are still running in non-P2P mode.

If you intend your relay to be part of a stake pool operation, then you need your relay to be well connected to the other relays of the Cardano network.  If it is not well connected, then blocks produced by your block-producer risk getting poorly propagated.  And, if your blocks are poorly propagated, then they risk not getting adopted into the chain which will result in a loss of staking rewards.

A notification service at api.clio.one has been setup to allow stake pool operators to share connection information about their relays.  To use this service, api.clio.one requires that you send your connection IP address and port number, as well as your current synchronisation status.

Create a cron job that will get executed every hour to ping api.clio.one with your connection information.  
Create file /etc/cron.d/cn-update-topology-s containing:  
```
PATH=/sbin:/bin:/usr/sbin:/usr/bin

# Every hour at 42mins past
#+---------------- minute (0 - 59)
#|  +------------- hour (0 - 23)
#|  |  +---------- day of month (1 - 31)
#|  |  |  +------- month (1 - 12)
#|  |  |  |  +---- day of week (0 - 6) (Sunday=0 or 7)
#|  |  |  |  |
42  *  *  *  *     root    systemd-cat -t 'cn-update-topology' cn-update-topology -s
```
Change the 42 value to whatever minute you wish.

Take a look at the /usr/bin/cn-update-topology script so that you understand how it works.  It is good practice to never blindly trust scripts written by other people!  The script defaults to using port 3001 and the hostname output by the 'hostname -f' command.

If you changed the port for your relay from the default of 3001 then you will need to configure it in /etc/cardano/my-cardano-node-config.json.  Eg:
```
{
  "port": 4001
}
```

Run the script manually once first to check the return status from api.clio.one
```
cn-update-topology -s
```
> { "resultcode": "201", "datetime":"2022-10-08 22:45:01", "clientIp": "27.99.14.36", "iptype": 4, "msg": "nice to meet you" }

If you run the script before your relay is fully synchronised to the blockchain you will see output like:
> { "resultcode": "503", "datetime":"2022-10-09 00:05:26", "clientIp": "27.99.14.36", "msg": "blockNo 5086230 seems out of sync. please retry" }

Make sure that the hostname your relay is using resolves to the external (public) IP address that api.clio.one will see.  Your external, or public, IP address is the address that other internet connected machines see when your machine connects with them.  Here is one way to check this:
```
curl -s https://icanhazip.com
```
> 27.99.14.36

Ensure that the hostname that you are feeding to api.clio.one for your relay resolves to this IP address.  If you are using your own domain name, you may need to update your DNS configuration.  You can use the linux tool 'dig' on your linux PC to check the resolution of your hostname on the internet:
```
dig @8.8.8.8 relay1.mydomainname.com A
```
> ;; ANSWER SECTION:
> relay1.mydomainname.com.	21600	IN	A	27.99.14.36

You can use the dig tool on your local linux pc to save installing the dnsutils package on your relay.

If you don't have your own domain name then you will need to set the name your ISP uses that resolves to your external IP address.  Check this value using the 'host' tool to do a reverse lookup on your external IP address.  Do this on your linux PC to save installing dnsutils on your relay:
```
host 27.99.14.36
```
> 36.14.99.27.in-addr.arpa domain name pointer n27-99-14-36.mrk1.qld.optusnet.com.au.

Obviously, use the IP address you received from the previous dig command.  Then double-check that the forward resolution for this name returns the same IP address:
```
host n27-99-14-36.mrk1.qld.optusnet.com.au
```
> n27-99-14-36.mrk1.qld.optusnet.com.au has address 27.99.14.36

So, in the above case, you could use the hostname 'n27-99-14-36.mrk1.qld.optusnet.com.au'.  You can configure this value to be sent by the cn-update-topology script by setting it in /etc/cardano/my-cardano-node-config.json:
```
{
  "port": 4001,
  "hostname": "n27-99-14-36.mrk1.qld.optusnet.com.au"
}
```
But it would be more professional to have your own proper DNS name for your relay if you are going to use it for your stake pool operation.  Setting up DNS is beyond the scope of this document.  (Note: You don't need the "port" setting above if you are using the default of 3001.)

Each time the cron job runs, your systemd journal will log the response from api.clio.one.  It should say saying someting like: "glad you're staying with us" to indicate success.  You can check this in your journal with:
```
journalctl --since today | grep 'cn-update-topology'
```
And you should see output like:
> Oct 09 08:47:03 relay1 cn-update-topology[190504]: { "resultcode": "204", "datetime":"2022-10-08 22:47:01", "clientIp": "27.99.14.36", "iptype": 4, "msg": "glad you're staying with us" }

## 8. Manually update mainnet-topology.json file periodically
Only do this step if you chose to run your relay in non-P2P mode.

If you are running this relay as part of a stake pool operation, then you need your relay to remain well connected to other relays.  The state of the Cardano network changes over time as relays come and go.  When running in P2P mode, your relay can find out about other current relays by itself.  However, if you are running your relay in non-P2P mode, then you will need to update the topology configuration from time to time so your relay can know some other current relays.

One way to update your mainnet-topology.json file is to manually edit it and configure the other relays that you want to connect with.  You can get a list of registered mainnet relays from: https://explorer.mainnet.cardano.org/relays/topology.json  You could just choose 10-20 and put them in your mainnet-topology.json file.

A more convenient method for updating your mainnet-topology.json file is to download one from api.clio.one.  api.clio.one provides a service which produces a topology configuration tailored for your geo-location (calculated from your IP address). But, you can only use this service after you have been pinging api.clio.one with your own relay information for at least 4 hours. (See above.)

The cn-update-topology script can be used to download a topology.json file from api.clio.one.  But, before using this script, edit /etc/cardano/my-cardano-node-config.json file.  Check that "port" is what you want and add any custom "peers" that you always want included.  Also configure the "hostname" if required. (See above.)  Eg:
```
{
  "port": 3001,
  "poolId": "#FIXME",
  "pooltoolApiKey": "#FIXME",
  "peers": [
    {
      "addr": "relay.myfirstfrienddomainname.com",
      "port": 3001,
      "valency": 1
    },
    {
      "addr": "relay.mysecondfrienddomainname.com",
      "port": 3001,
      "valency": 1
    }
  ]
}
```

Use cn-update-topology script to download a topology.json file from api.clio.one as follows:
```
cn-update-topology -f 15
```
You can have the output of the above command overwrite the existing mainnet-topology.json file.  If you use this method, be careful to always check the contents of mainnet-topology.json file.  The following command will overwrite mainnet-topology.json and immediately show its new contents:
```
cn-update-topology -f 15 > /etc/cardano/mainnet-topology.json && cat /etc/cardano/mainnet-topology.json
```

Restart your cardano-node in order to use the new topology file:
```
systemctl restart cardano-node
```

