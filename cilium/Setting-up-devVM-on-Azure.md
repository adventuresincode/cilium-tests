## Install docker-io on Ubuntu
```sh
sudo apt install docker.io
sudo usermod -aG docker $USER
```

## Install kubernetes tools
### Install kubectl
Download    
```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```
Validate the downloaded binary:   
1. Download the checksum:
```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
```
2. Validate the binary against the checksum
```sh
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
``` 
3. Install kubectl:   
```sh
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
## Install KIND on Ubuntu
```sh
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```