# Celestia Light Node (blockspace-race) Kurulum Rehberi

![Ekran görüntüsü 2023-03-28 040506](https://user-images.githubusercontent.com/94050636/228100509-865911b0-e167-40e5-9049-7538e0017276.png)


## Minimum Önerilen Sistem Gereksinimleri

Ubuntu Linux 20.04 (LTS) <br/>
Memory: 2 GB RAM <br/>
CPU: Single Core <br/>
Disk: 5 GB SSD Storage <br/>
Bandwidth: 56 Kbps for Download/56 Kbps for Upload <br/>


## Sistem güncellemerini yaparak başlıyoruz.

```
sudo apt update && sudo apt upgrade -y

sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential \
git make ncdu -y
```

## Go kurulumu ile devam ediyoruz.

```
ver="1.20.3"
cd $HOME
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```
go versiyon komutunun çıktısı şu şekilde olmalıdır:

```
go version go1.20.3 linux/amd64
```

## celestia-node Kurulumu

```
cd $HOME
rm -rf celestia-node
git clone https://github.com/celestiaorg/celestia-node.git
cd celestia-node/
git checkout tags/v0.10.0
make build
make install
make cel-key
```

Celestia versiyon kontrolü:

```
celestia version
```

Çıktı şu şekilde olmalı:

```
Semantic version: v0.10.0
```

## Node'u başlatalım

Not: Aşağıdaki komutu girdiğinizde my_celes_key adında cüzdan oluşacak. Cüzdan adresi ve mnemonicler çıkacak. Bunları kaydetmeyi unutmayın!!!

```
celestia light init --p2p.network blockspacerace
```

Şuna benzer bir çıktı almalısınız:

```
INFO    node    nodebuilder/init.go:29  Initializing Light Node Store over '/root/.celestia-light-blockspacerace-0'
INFO    node    nodebuilder/init.go:61  Saved config    {"path": "/root/.celestia-light-blockspacerace-0/config.toml"}
INFO    node    nodebuilder/init.go:63  Accessing keyring...
WARN    node    nodebuilder/init.go:135 Detected plaintext keyring backend. For elevated security properties, consider using the `file` keyring backend.
INFO    node    nodebuilder/init.go:150 NO KEY FOUND IN STORE, GENERATING NEW KEY...    {"path": "/root/.celestia-light-blockspacerace-0/keys"}
INFO    node    nodebuilder/init.go:155 NEW KEY GENERATED...

NAME: my_celes_key
ADDRESS: celestiaxxxxxx
MNEMONIC (save this somewhere safe!!!): 
hand change flame prepare satisfy xxxx xxxx xxx xxxx xxxx
```

## Cüzdan adresi ve adını şu kod ile görüntüleyebilirsiniz:

```
./cel-key list --node.type light --p2p.network blockspacerace
```

Cüzdan adresinize Celestia Discord sunucusundaki Blockspace Race başlığı altında bulunan #faucet kanalından test coini almayı unutmayın. Cüzdanda coin olması gereklidir.


## Servis oluşturmayla devam ediyoruz.

```
sudo tee <<EOF >/dev/null /etc/systemd/system/celestia-lightd.service
[Unit]
Description=celestia-lightd Light Node
After=network-online.target

[Service]
User=$USER
ExecStart=/usr/local/bin/celestia light start --core.ip https://rpc-blockspacerace.pops.one --core.rpc.port 26657 --core.grpc.port 9090 --keyring.accname my_celes_key --metrics.tls=false --metrics --metrics.endpoint otel.celestia.tools:4318 --gateway --gateway.addr localhost --gateway.port 26659 --p2p.network blockspacerace
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```

Not: ExecStart komutundaki core ip, rpc port, grpc port, [Celestia dökümanlarında](https://docs.celestia.org/nodes/blockspace-race/#rpc-endpoints) sağlanan endpointlerden alınmıştır ve alternatifleri vardır. İsterseniz burayı değiştirip farklı endpointler kullanabilirsiniz.

Aşağıdaki komutlarla node'umuzu başlatıyoruz.

```
systemctl enable celestia-lightd
systemctl start celestia-lightd
```

Log kontrolü

```
journalctl -u celestia-lightd.service -f
```

Senkronize olduktan sonra loglar şu şekilde gözükmeli;

```
INFO    header/store    store/store.go:349      new head        {"height": 106151, "hash": "CD084A85460978EC8C9E4BA23EE3847B28A90A879245E46F68B8133E413EA7A3"}
INFO    das     das/subscriber.go:34    new header received via subscription    {"height": 106151}
INFO    das     das/worker.go:79        finished sampling headers       {"from": 106151, "to": 106151, "errors": 0, "finished (s)": 0.000060704}
WARN    header/sync     sync/sync_head.go:140   received known network header   {"current_height": 106151, "header_height": 106151, "header_hash": "CD084A85460978EC8C9E4BA23EE3847B28A90A879245E46F68B8133E413EA7A3"}
INFO    header/store    store/store.go:349      new head        {"height": 106152, "hash": "C0FA44EF29FDE75ECD069A2031CC0A8BE94996F775FC106F76B88CCA514B1EF9"}
INFO    das     das/subscriber.go:34    new header received via subscription    {"height": 106152}
INFO    das     das/worker.go:79        finished sampling headers       {"from": 106152, "to": 106152, "errors": 0, "finished (s)": 0.00005358}
```

## Node ID öğrenme

Not: Aşağıdaki iki komutu girerek node ID'mizi öğreneceğiz. Bu node ID ile Celestia Light Node'umuzu [Tiascan](https://tiascan.com/light-nodes) üzerinden uptime gibi çeşitli verilerle birlikte takip edebileceğiz. Ayrıca Knack Portal'da tasklarda node ID'mizi girmemiz gerekecek. Bu ID ile node'umuz takip edilecek, çalışma süresi izlenecek.

```
AUTH_TOKEN=$(celestia light auth admin --p2p.network blockspacerace)


curl -X POST \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H 'Content-Type: application/json' \
     -d '{"jsonrpc":"2.0","id":0,"method":"p2p.Info","params":[]}' \
     http://localhost:26658
```

Bu iki komutu girdiğinizde şuna benzer bir çıktı almalısınız;

```
{"jsonrpc":"2.0","result":{"ID":"12D3KooXXXXXX",..........................
```

12D3 ile başlayan node'umuzun ID'sidir. [Tiascan](https://tiascan.com/light-nodes) üzerinden aratabilirsiniz.

## Yedeklenmesi Gereken Dosyalar

WinSCP veya aynı işleve sahip bir araç ile .celestia-light-blockspacerace-0 klasörünü altında bulunan keys klasörünü yedeklemeniz gerekmektedir. Key bilgilerimiz burada. Knack portala node bilgilerinizi kaydetmeden önce bu klasörü mutlaka yedekleyin!


## Diğer servis komutları

Servis Kontrolü

```
systemctl status celestia-lightd
```

Restart:

```
systemctl restart celestia-lightd
```

Durdurma:

```
systemctl stop celestia-lightd
```

## BAŞARILAR!
