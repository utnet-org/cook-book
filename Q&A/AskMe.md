# TroubleShooting

**Q: 是不是只需要部署miner server和worker就行，node不需要部署？**

A: 部署node作为全节点使用。unc node应该部署在miner机器上。

**Q: miner server和worker启动有没有先后顺序？**

A: 启动顺序应该没有依赖关系，最好使用守护模式。例如，使用pm2守护node。建议先部署worker，然后部署unc-node和miner。

**Q: worker启动后暴露的端口是多少？配置在cloudflare tunnel里面，然后才能在启动worker的时候指定这个ip。**

A: 端口是7001。

**Q: miner server在启动的时候，--node=addr0，这个node地址是什么？**

A: node地址就是testnet/mainnet的地址，unc-node的3030端口。

**Q: worker server启动的时候，--service ip是写区域网的ip还是写被cloudflare代理的域名地址？**

A: 要通信的话，填写代理后的cf host api地址，使用cf tunnel打洞。

**Q: key文件放在worker机的哪个目录，需不需要做什么操作？**

A: key文件放在`bm_chip/src/key`目录下，不需要进行任何操作，只需等待命令读取。

**Q: worker的服务端口是什么？server服务是否需要开启一些来自外部的安全组入站规则？**

A: worker的服务端口是3030。node需要开放TCP端口12345和3030，12345作为对等节点的所有入站流量，3030可以考虑加入白名单。

**Q: `$ sudo go build -o ../workerserver no Go files in /opt/uminer/miner-server`报错了。**

A: 先执行`go mod tidy`。

**Q: `protoc --go_out=. --go-grpc_out=. ./chip.proto`报错，提示没有安装protoc。**

A: 执行`go install google.golang.org/protobuf/cmd/protoc-gen-go`，然后设置路径：`export PATH=$PATH:$GOPATH/bin`。在各自目录下，对应的proto文件都执行一遍。

**Q: unc-node、unc-cli和miner server/worker的关系是什么？**

A: unc-node和unc-cli是节点的一部分，属于激励层面的链。miner通过调用worker、unc-node和unc-cli来完成任务。

**Q: 执行`cargo install unc-validator`报错了。**

A: 执行`sudo apt-get install make`。

**Q: minerserver启动的时候指定了worker IP，node的地址就是这个unc-node的地址吗？**

A: 在启动minerserver时，使用`--node`参数指定的是unc-node的地址，端口为3030。
