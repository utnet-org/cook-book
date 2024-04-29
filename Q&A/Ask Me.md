## Question 

Q: 是不是只需要部署miner server和worker就行，node不需要部署
A: 部署node, 作为全节点使用, unc node部署在那里？server机器还是worker机器？ 在miner

Q: miner server和worker启动有没有先后顺序
A: 启动顺序应该没有依赖关系， 最好守护模式， 如node 就是pm2 守护, 建议先部署worker，然后部署unc-node，miner
miner和worker是两个不同的机器，但建议先起worker，不然miner起了有些服务需要worker，就找不到worker

Q: worker启动后暴露的端口是多少？配置在cloudflare tunnel里面，然后才能在启动worker的时候指定这个ip
A: 7001

Q: miner server在启动的时候，--node=addr0，这个node地址是什么
A: node地址就是 testnet/mainnet那个地址, unc-node 的3030端口

Q: worker server启动的时候，--service  ip是写区域网的ip还是写被cloudflare代理的域名地址
A: 要通信，填写代理后cf host api地址, 使用cf tunnel 打洞

Q: key文件放在worker机的那个目录，需不需要做什么操作
A: 在bm_chip/src/key.   可以不用任何操作，就等着被命令读就可以

Q: woker的服务端口是什么，server服务是否需要开启一些来自外部的安全组入站规则
A: node 需要 tcp 12345 3030 两个端口开放, 12345作为对等节点 all入站, 3030可以考虑加白名单

Q: $ sudo go build -o ../workerserver no Go files in /opt/uminer/miner-server
A: 先go mod tidy一下

Q: protoc --go_out=. --go-grpc_out=. ./chip.proto, 没有安装protoc
A: go install google.golang.org/protobuf/cmd/protoc-gen-go && chmod +x $GOPATH/bin/protoc-gen-go && 设置 path：export PATH=$PATH:$GOPATH/bin
在各自目录下，protoc各自的proto 都执行一遍

Q: unc-node、unc-cli和miner server/worker的关系?
A: unc-node / cli 是节点的， 激励层面的， 总的来说是上层的链， 激励层作用, miner去call worker unc-node unc-cli

Q: cargo install unc-validator这一步骤执行报错了
A: sudo apt-get install make

Q: minserver启动的时候指定了worker IP，node的地址就是这个unc-node的地址么？
A: 3030 端口在启动minerserver的时候node参数