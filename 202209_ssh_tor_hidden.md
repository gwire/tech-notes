# Connecting to ssh as a Tor v3 Hidden Service

I have two servers on two different networks, `blue` and `green`, that need to communicate with each other over over ssh.

Rather than configuring these to allow connections on a public IP address, I can configure the ssh service as a “v3” hidden network service on the Tor network.
Furthermore, I can use client auth tokens for the connection to ensure that only Tor clients with a specific key are able to connect. (The keys in this case aren’t replacing ssh’s authentication methods, more analogous to locking down connections to specific IP ranges.)

(Note that client keys are applied to all hidden services on a server, so they may not be appropriate for something that’s running any public onion services.)

This is an overview for setting up a connection from a client (green) to a server (blue) on Ubuntu (some details may be different on other platforms).

## Generating a keypair

On the client (green):

1. Generate a public and private key, for example via the method given in [Tor’s Client Auth guide](https://community.torproject.org/onion-services/advanced/client-auth/):

    ```shell-session
    green$ cd /tmp
    green:/tmp$ openssl genpkey -algorithm x25519 -out green.prv.pem
    green:/tmp$ cat green.prv.pem | grep -v " PRIVATE KEY" | base64pem -d | tail --bytes=32 | base32 | sed 's/=//g' > green.prv.key
    green:/tmp$ openssl pkey -in green.prv.pem -pubout | grep -v " PUBLIC KEY" | base64pem -d | tail --bytes=32 | base32 | sed 's/=//g' > green.pub.key
    ```

2. Note the results (and store the resulting strings in a password manager)

    ```shell-session
    green:/tmp$ cat green.prv.key 
    5A2IP5UUIUWKMMHFDPHIZEC5Z777Z4OGXSOXCAMWSL4VCV5TWZVA
    green:/tmp$ cat green.pub.key 
    HSO6MUFK7V3HGD4CTTWBX5WSZL5SY6HZIAPFOALVTKMK423PG5DA
    ```

    ***Don't use the example keys shown above!***


## Configuring a hidden ssh service

On the server (blue):

1. First, check that localhost can conect to a running ssh service:

    ```shell-session
    blue$ echo | nc 127.0.0.1 22
    SSH-2.0-OpenSSH_8.9p1
    Invalid SSH identification string.
    ```

1. If the connection is rejected, and if TCP Wrappers is in use, the following may need to be added to `/etc/hosts.allow`

    ```
    sshd: 127.0.0.1
    sshd: [::1]/128
    ```

1. Add the following into `/etc/tor/torrc`

    ```
    HiddenServiceDir /var/lib/tor/hidden_service/ 
    HiddenServicePort 22 127.0.0.1:22
    ```

1. Reload tor.

    ```shell-session
    blue$ sudo systemctl reload tor
    ```

    This will create the directory `/var/lib/tor/hidden_service/authorized_clients`

1. Create the file `/var/lib/tor/hidden_service/authorized_clients/green.auth`
that consists of `descriptor:x25519:` and the content of `green.pub.key`

    ```
    descriptor:x25519:HSO6MUFK7V3HGD4CTTWBX5WSZL5SY6HZIAPFOALVTKMK423PG5DA
    ```

1. Reload tor again.

    ```terminal-session
    blue$ sudo systemctl reload tor
    ```

1. Make a note of the `.onion` name

    ```terminal-session
    blue$ sudo cat /var/lib/tor/hidden_service/hostname 
    y52ebqgutuoiimirpcrgffsf5eu5withbuvustutheppqzm6tvyzlwid.onion
    ```

    ***Don’t use the example name shown above!***

## Setting up the ssh client to use a hidden service


On the client (green):

1. Test that the SOCKS 5 connection breaks without the client key in place:

     ```terminal-session
     green$ echo | nc -X 5 -x localhost:9050 y52ebqgutuoiimirpcrgffsf5eu5withbuvustutheppqzm6tvyzlwid.onion 22
     nc: connection failed, SOCKSv5 error: General SOCKS server failure
     ```

1. Create the directory for storing the client authentication keys:

    ```terminal-session
    green$ sudo mkdir /var/lib/tor/onion_auth
    green$ sudo chown debian-tor /var/lib/tor/onion_auth
    ```

    (The owner needs to be the user tor runs as, `debian-tor` on deb-based systems.)

1.  Create the file `/var/lib/tor/onion_auth/blue.auth_private` consiting of a single line that includes the hidden service hostname, `:descriptor:x25519:`, and the content of `green.prv.key`

    ```
    y52ebqgutuoiimirpcrgffsf5eu5withbuvustutheppqzm6tvyzlwid:descriptor:x25519:5A2IP5UUIUWKMMHFDPHIZEC5Z777Z4OGXSOXCAMWSL4VCV5TWZVA
    ```

1. Specify the client key directory in `/etc/tor/torrc`

    ```
    ClientOnionAuthDir /var/lib/tor/onion_auth
    ```

1. Reload tor

    ```terminal-session
    green$ sudo systemctl reload tor
    ```

1. Check that a SOCKS 5 connection via Tor now returns the SSH banner:

    ```terminal-session
    green$ echo | nc -v -X 5 -x localhost:9050 y52ebqgutuoiimirpcrgffsf5eu5withbuvustutheppqzm6tvyzlwid.onion 22
    SSH-2.0-OpenSSH_8.9p1
    Invalid SSH identification string.
    ```

1. add the following to `~/.ssh/config` on green:

    ```
    Host *.onion
      ProxyCommand nc -X 5 -x localhost:9050 %h %p
    
    Host blue.onion
      Hostname y52ebqgutuoiimirpcrgffsf5eu5withbuvustutheppqzm6tvyzlwid.onion
    ```

1. Test the connection (assuming sshd is configured to accept passwords):

    ```terminal-session
    green$ ssh admin@blue.onion
    admin@y52ebqgutuoiimirpcrgffsf5eu5withbuvustutheppqzm6tvyzlwid.onion's password: 
    ```
 
    (You can now add further certificate/agent details to `~/.ssh/config`)
