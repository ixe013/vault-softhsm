# Vault with SoftHSM

This is a manual setup to sandbox a Vault Enterprise+HSM setup with SoftHSMv2, an open source software-based HSM that uses the PKCS#11 standard interface. 

You need sudo access to install the required packages, but everything else runs in your own account.


# Up and running

## Install Vault

```
curl -OL -C - https://releases.hashicorp.com/vault/1.5.3+ent.hsm/vault_1.5.3+ent.hsm_linux_amd64.zip
sudo unzip vault_*ent.hsm_linux_amd64.zip -d /usr/local/bin/.
```

## Install SoftHSM

The required packages for Vault HSM on Debian 9 are libltdl7, libsofthsm2, and softhsm2; install them like this:

```
chmod +x run.sh
sudo bash ./run.sh install
```

## Configure the clusther

The cluster will run on 127.0.0.1:8200 by default. To change the port, set the `API_PORT`environment variable. You can also
set the `CLUSTER_PORT` variable, or it will default to `API_PORT+1`. For example:

```
export API_PORT=9200
```

The `run.sh` script will configure everything for you, just run:

```
./run.sh config
```

You can clean the previous keys by running a `clean` first, like this:

```
./run.sh clean config
```

## Start the cluster

```
./run.sh start
```

## Stop the cluster

```
./run.sh stop
```


# Manually installing everything

It's also quite handy to have PKCS related tools such as pkcs11-tool when debugging issues; you can optionally install this tool as part of the opensc package:

```
$ sudo apt-get install -y opensc
```

The SoftHSM PKCS#11 shared library will be installed to /usr/lib/softhsm by default on Debian, and you can do a quick verification of its presence to ensure that it's where we expect it to be:

```
$ ls -lh /usr/lib/softhsm/
total 2.7M
-rw-r--r--.  1 root root 691K Feb 12  2017 libsofthsm2.so
```

Here we note that our Cryptoki shared library file libsofthsm2.so; your file could be named either libsofthsm.so or softhsm.so and be located elsewhere depending on OS distribution.

Make sure that you locate it and note its full path for later use.

### Configure SoftHSM to work with Vault
Now that you have the required software installed, you'll need to establish a minimal SoftHSM configuration and initialize a token in the slot that you'll use with Vault; first create a directory for the SoftHSM data:

```
export SOFTHSM_CONF=$HOME/.config/softhsm2/softhsm2.conf
mkdir -p -v $HOME/softhsm/tokens/ $HOME/.config/softhsm2/ $HOME/.config/vault/1 $HOME/.config/vault/2 $HOME/.config/vault/3
```

Next, create a basic SoftHSM configuration file:

```
tee $HOME/.config/softhsm2/softhsm2.conf <<EOF
# SoftHSM v2 configuration file
directories.tokendir = $HOME/softhsm/tokens/
objectstore.backend = file
# ERROR, WARNING, INFO, DEBUG
log.level = DEBUG
EOF
```

Using the softhsm2-util tool, let's now take a moment to examine the current state of our shiny new software based HSM:

```
$ softhsm2-util --show-slots
Available slots:
Slot 0
    Slot info:
        Description:      SoftHSM slot ID 0x0
        Manufacturer ID:  SoftHSM project
        Hardware version: 2.2
        Firmware version: 2.2
        Token present:    yes
    Token info:
        Manufacturer ID:  SoftHSM project
        Model:          SoftHSM v2
        Hardware version: 2.2
        Firmware version: 2.2
        Serial number:
        Initialized:      no
        User PIN init.:   no
        Label:
```

Let's use softhsm2-util to initialize a new token in the next free slot for use with Vault:

```
$ 
declare -A VAULT_HSM_SLOTS=()
for N in 1 2 3 
do
    softhsm2-util --init-token --free --label "vault-hsm${N}" --pin 1234 --so-pin asdf
    VAULT_HSM_SLOT[$N]=$(softhsm2-util --show-slots | grep ^Slot | sed "${N}q;d" | cut -d\  -f2)
done
```

Now our slots look like this, with the newly initialized slot 0, which is now actually referred to as slot 112253752 but with the same hsm_example label we originally defined, and the original empty slot 0 is now occupying the slot 1 location:

```
$ softhsm2-util --show-slots
Slot 112253752
    Slot info:
        Description:      SoftHSM slot ID 0x6b0db38
        Manufacturer ID:  SoftHSM project
        Hardware version: 2.2
        Firmware version: 2.2
        Token present:    yes
    Token info:
        Manufacturer ID:  SoftHSM project
        Model:          SoftHSM v2
        Hardware version: 2.2
        Firmware version: 2.2
        Serial number:    2fb8da6586b0db38
        Initialized:      yes
        User PIN init.:   yes
        Label:          hsm_example
Slot 1
    Slot info:
        Description:      SoftHSM slot ID 0x1
        Manufacturer ID:  SoftHSM project
        Hardware version: 2.2
        Firmware version: 2.2
        Token present:    yes
    Token info:
        Manufacturer ID:  SoftHSM project
        Model:          SoftHSM v2
        Hardware version: 2.2
        Firmware version: 2.2
        Serial number:
        Initialized:      no
        User PIN init.:   no
```

So that looks good; note the new slot value as we'll need that when configuring Vault.

### Vault Enterprise + HSM Configuration

We'll be using the most minimal non-developer-mode Vault configuration possible for this guide, which is to run the Vault server with the file storage backend for all Vault data, while also specifying the required HSM configuration.

Here's an example minimal Vault HSM configuration; make sure to use the correct value for slot as shown in the `softhsm2-util --show-slots` command output and the correct path your libsofthsm2.so file.

```
$ 
for N in 1 2 3 
do
cat << EOF > $HOME/.config/vault/${N}/config.hcl
listener "tcp" {
  address = "127.0.0.1:8200"
  tls_disable = "true"
}

storage "raft" {
  path = "$HOME/.config/vault/${N}"
  node_id = "data-vault-${N}"
}

#I am testing on WSL which does not support mlock
disable_mlock = true

ui = true
api_addr = "https://127.0.0.${N}:8200"
cluster_addr = "https://127.0.0.${N}:8201"

seal "pkcs11" {
  lib            = "/usr/lib/softhsm/libsofthsm2.so"
  slot           = "${VAULT_HSM_SLOT[$N]}"
  pin            = "1234"
  key_label      = "vault-hsm${N}"
  hmac_key_label = "hmac-key"
  generate_key   = "true"
}
EOF
done
```

Start Vault with the following command line :

```
$ 
for N in 1 2 3 
do
    nohup vault server --config $HOME/.config/vault/${N}/config.hcl --log-level=trace 2>&1 ;#> vault${N}.log &
done
tail -50f vault*.log
```

Initialize Vault using a special set of command line flags for use with HSM:

```
$ export VAULT_ADDR=http://localhost:8200
$ vault operator init -recovery-shares=1 recovery-threshold=1
Recovery Key 1: A63tTM9+OzxA9t1x5JOJ4WoDJt3xCQDUaiLgHin3xXI=

Initial Root Token: s.M5NklwwKZgyBezgJ6oLFuXJH

Success! Vault is initialized
```

You’ll note that instead of an unseal key, a recovery key is issued when using an HSM.
