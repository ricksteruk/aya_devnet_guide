## Updating your existing Aya Node

Most recent version:
[Release DevNet AyA Node v0.3.0](https://github.com/worldmobilegroup/aya-node/releases/tag/devnet-v0.3.0)

To check what version of aya-node your machine is running:
```bash
cd /home/${USER}/aya-node
./target/release/aya-node --version
```

### Running the update
We will shut down the aya-node service, delete the old files, download the new chainspec and aya-node binary files and then restart the aya-node

NOTE: `${USER}` is a global variable that returns your username.  You do not need to replace this!


```bash
sudo systemctl stop aya-node.service
cd /home/${USER}/aya-node
rm -r wm-devnet-chainspec.json
wget https://github.com/worldmobilegroup/aya-node/releases/download/devnet-v0.3.0/wm-devnet-chainspec.json
rm -r target/release/aya-node
wget -P target/release https://github.com/worldmobilegroup/aya-node/releases/download/devnet-v0.3.0/aya-node
chmod +x target/release/aya-node
sudo systemctl restart aya-node.service
```

To check your aya-node was sucessfully updated:
```bash
cd /home/${USER}/aya-node
./target/release/aya-node --version
```
