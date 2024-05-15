# 一. 环境准备

我们的节点基于Ubuntu 2004/2204 X86-64系统, 16core, 500G最低配置 下面的操作都是在此系统下部署

## 1. 部署机器

部署相关工作

```sh

# miner-mach 机器预装node.js
# 使用 curl 下载 Node.js 18.x or beyond 安装脚本
curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -

# 安装 Node.js
sudo apt-get install -y nodejs

# 安装 pm2
npm install -g pm2

# 安装cargo
curl https://sh.rustup.rs -sSf | sh
rustc --version

```

we get something like this:

![rust](../../images/rust.png)

+ 安装常用的系统工具：git 、curl、wget、build-essential
+ awk/df一般都是默认安装的，其它工具可使用以下命令安装：

```sh
sudo apt install git curl wget build-essential -y
```

+ 安装openssl依赖

```sh

sudo apt install pkg-config libssl-dev libclang-dev -y
sudo apt-get install --assume-yes libudev-dev
```

+ 安装unc cli 和 unc-node

```sh
# 安装 unc-cli 工具
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/utnet-org/utility-cli-rs/releases/download/v0.10.2/utility-cli-rs-installer.sh | sh

# 安装验证者 validator-cli 工具
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/utnet-org/utility-validator-cli-rs/releases/download/v0.10.2/unc-validator-installer.sh | sh

# 下载unc-node 节点, 如 2004/2204 任意适合包
wget -O - https://github.com/utnet-org/utility/releases/download/v0.12.1/x86_64-ubuntu-2004-unc-node.tar.gz | tar -xz

or

wget -O - https://github.com/utnet-org/utility/releases/download/v0.12.1/x86_64-ubuntu-2204-unc-node.tar.gz | tar -xz


# 解压得到 unc-node 二进制文件, 放置在/opt/unc-node/, 后面有用
sudo mkdir /opt/unc-node
sudo chmod 777 /opt/unc-node
cp unc-node  /opt/unc-node/

```

we get something like this:

![unc-node](../../images/node.png)

## 2. 机器的标准配置

+ 确保libc的版本，可以运行我们rust编译出来的二进制程序

```sh
ldd --version
ldd (Ubuntu GLIBC 2.35-0ubuntu3.7) 2.35
Copyright (C) 2022 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.
```

+ 机器上需要正确安装node(可shell正确执行node和npm命令)， version 18

```sh
node -v
v21.7.3
```
  
# 二. 配置文件的编写

***这个是新矿工节点的关键步骤***  
下面创建新的描述目录，并在里面增加对应的配置文件

```sh

# testnet node init directly use binaries
/opt/unc-node/unc-node  --home /opt/unc-node  init --chain-id testnet --download-genesis --download-config

# download snapshot data （optional）
## install rclone 1.66.0 or beyond
## Linux
$ sudo apt install rclone

$ mkdir -p ~/.config/rclone
$ touch ~/.config/rclone/rclone.conf

## rclone.conf
[unc_cf]
type = s3
provider = Cloudflare
endpoint= https://ec9b597fa02615ca6a0e62b7ff35d0cc.r2.cloudflarestorage.com
access_key_id = 2ff213c3730df215a7cc56e28914092e
secret_access_key = b28609e3869b43339c1267b59cf25aa5deff4097737d3848e1491e0729c3ff6c
acl = public-read

## download data 
$ rclone copy --no-check-certificate unc_cf:unc/latest ./
$ latest=$(cat latest)
$ rclone copy --no-check-certificate --progress --transfers=6  unc_cf:unc/${latest:?}.tar.gz /tmp

$ 解压快照到/opt/unc-node/data
tar -zxvf /tmp/${latest:?}.tar.gz -C /tmp  && mv /tmp/${latest:?}/data /opt/unc-node

```

we get something like this:

![rust](../../images/node-init.png)

# 三. 创建账户并转账

## 使用 unc cli 创建账户/转账/添加key操作

```sh
## create new accounts 用例, 使用自己创建的account id 而不是用例里面的`fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43`
unc account create-account fund-later use-auto-generation save-to-folder ~/.unc-credentials/implicit
## as follows:
## fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43.json

## 通过unc基金会账户或者水龙头 给新创建的账户一些资助(pledge 质押需要unc 代币)`fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43`  100K unc coin
unc tokens unc send-unc fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43 '100000 unc' network-config testnet sign-with-keychain send

## 在 /opt/unc-node 目录下 执行`touch validator_key.json` 命令, 验证者信息, 从上面的创建的账户~/.unc-credentials/implicit 信息填写即可  (optional)

{
    "account_id": "fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43"
    "public_key":"ed25519:EYM66gAFekoEgLjicPyvZQiFRrbvNsWjVDQQ11fASV3Q",
    "private_key":"ed25519:3NVx4sHxBJciEH2wZoMig8YiMx1Q84Ur2RWTd2GQ7JNfWdyDxwwYrUR6XtJR3YcYeWh9NzVEmsnYe2keB97mVExZ"
}
```

we get something like this:

![create-account](../../images/create-account.svg)

# 四. 正式部署服务和配置

## pre-binaries 安装方式

## 1. 部署脚本

+ unc-node.ecosystem.config.js 文件内容如下:
  其中unc-node 上面的下载节点路径

```js
module.exports = {"apps":[{"name":"unc-node","script":"/opt/unc-node/unc-node","env":{"HOME":"/opt/unc-node"},"exec_mode":"fork","watch":"false","autorestart":true,"restart_delay":5000,"cwd":"/opt/unc-node","args":" --home=/opt/unc-node run"}]}
```

we get something like this:

![pm2](../../images/pm2.svg)

## 2. 服务启动和关闭

```sh
# 运行节点
pm2 start unc-node.ecosystem.config.js

# 停止节点
pm2 stop unc-node.ecosystem.config.js

# 查看日志
pm2 logs

# 更多命令或GPT pm2
pm2 --help
```

通过pm2 list来查看服务的运行状态  
比如在miner-mach1上，pm2 list显式如下

┌────┬─────────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id │ name        │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├────┼─────────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 0  │ unc-node    │ default     │ N/A     │ fork    │ 466460   │ 91s    │ 0    │ online    │ 0%       │ 154.5mb  │ ubuntu   │ enabled  │
└────┴─────────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘

可见这台机器上的unc-node服务都正常启动了，并且运行正常，没有反复重启

通过pm2 logs 确保unc-node服务已经正确启动了,  链同步快照都下载完成或者同步完成, 没有error级别日志, 正常打印区块高度
we get something like this:

![node-log](../../images/node-log.svg)

## Docker 安装方式

### Run the image from the command line

```sh
# Set node store location
export UNC_HOME=$HOME/.unc

# Set the node type
export CHAIN_ID=testnet

# Only Set Once Time init node  after `unset INIT`
export INIT=true

docker run \
    -v $HOME/node-store:$UNC_HOME \
    -e CHAIN_ID=$CHAIN_ID \
    -e UNC_HOME=$UNC_HOME \
    -e INIT=$INIT \
    -p 3030:3030 -p 12345:12345 \
    --name unc-node \
    ghcr.io/utnet-org/utility:latest
```

Or

### docker-compose.yml

```yaml
version: "3.8"
services:
  unc-node:
    image: ghcr.io/utnet-org/utility:latest
    environment:
      - UNC_HOME=$HOME/.unc
      - CHAIN_ID=testnet
      - INIT=true
    volumes:
      - ${HOME}/node-store:$HOME/.unc
    ports:
      - 3030:3030
      - 12345:12345
```

```sh
# 查看services
docker-compose up -d
docker ps
docker logs -f unc-node
docker exec -it <container-id> /bin/bash
```

## 3. 发起质押(前提是有token的账户)

```sh
# 直接发起质押, 100K unc
unc-validator pledging pledge-proposal fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43 ed25519:EYM66gAFekoEgLjicPyvZQiFRrbvNsWjVDQQ11fASV3Q '100000 UNC' network-config testnet sign-with-keychain send

# 查看直接质押金额
unc-validator pledging view-pledge fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43 network-config testnet now

# view tx status 查看tx 状态值 Success才行
unc transaction view-status EWHzhriCRTbDVd9SH6Vk88hSHqzJ7pipXW6eUhWTBkvS network-config testnet

```

## 4.注册矿工上链(只有基金会有权限操作)

```sh
## 使用基金会账户unc 注册Rsa2048 keys, ***batch_register.json.sample是矿工rsa keys文件, unc.json 是基金会unc账户 keys文件***  
unc extensions register-rsa-keys unc use-file ~/keys/batch_register.json.sample with-init-call network-config custom sign-with-access-key-file ~/.unc-credentials/unc.json send
```

## 添加 rsa public key 到验证者账户, 如`fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43`

```sh
unc account add-key fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43 grant-full-access use-manually-provided-public-key rsa2048:2TuPVgMCHJy5atawrsADEzjP7MCVbyyCA89UW6Wvjp9HrAzK4ZdyEHxs7qVnPrrF3R6w3zNJoHz828bNNJq9f6FPqyq9hsRP47qNXu1mcxeqWmRr8TKTBMLQNzNcjZVm6qX2BiSdXetAZsjPBMC6TyC2smee4s5Mqc4uh5rc5v7Z6nHWGxttHbhHGUCyWtgNGgPevFB2odsTdaXgcgWKtR3zLD6qrbaw631yNEJhverkLMrQJz436L21JWkgXpcTDRYPNWnk7DbztgA6RcgLmve3EG125eW2c2Bj7DkkWAVeWHZnXboDM8kYhAEbfRqUuKwn9K1m9adMqfig4xmM5wxGGABu5dD1gmthQRytLF1y3o2kpTtgrsNyBVTkqV7eMR9qJhUxwiU1rXdQKJ network-config testnet sign-with-keychain send
```

## 5.发起链上挑战(矿工领取算力, 周期性发起挑战)

```sh
# ***challenge.json.sample是挑战信息, rsa_signer_key.json是rsa keys文件, 这个是模拟的, 真实的得由算力机器发起***
unc extensions create-challenge-rsa fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43 use-file ~/keys/challenge.json.sample without-init-call network-config custom sign-with-access-key-file ~/keys/rsa_signer_key.json send

```

+ batch_register.json.sample 注册矿工配置信息,  字段`power` 默认单位是Tera即10.pow(12)

```json
[
    {
    "miner_id": "miner0",
    "public_key": "rsa2048:2TuPVgMCHJy5atawrsADEzjP7MCVbyyCA89UW6Wvjp9HrAzK4ZdyEHxs7qVnPrrF3R6w3zNJoHz828bNNJq9f6FPqyq9hsRP47qNXu1mcxeqWmRr8TKTBMLQNzNcjZVm6qX2BiSdXetAZsjPBMC6TyC2smee4s5Mqc4uh5rc5v7Z6nHWGxttHbhHGUCyWtgNGgPevFB2odsTdaXgcgWKtR3zLD6qrbaw631yNEJhverkLMrQJz436L21JWkgXpcTDRYPNWnk7DbztgA6RcgLmve3EG125eW2c2Bj7DkkWAVeWHZnXboDM8kYhAEbfRqUuKwn9K1m9adMqfig4xmM5wxGGABu5dD1gmthQRytLF1y3o2kpTtgrsNyBVTkqV7eMR9qJhUxwiU1rXdQKJ",
    "power": 100,
    "sn": "serial number here c0",
    "bus_id": "bus info c0",
    "p2key": "p2keys1 c0"
    },
    {
    "miner_id": "miner1",
    "public_key": "rsa2048:2TuPVgMCHJy5atawrsADEzjP7MCVbyyCA89UW6Wvjp9HrCGrtsUG9dcY8iaDtbnNTuA8PiVhpLgeiznHF3SUjAdYk4bSnwTShu5oNY5o3z1LYcgjiZhAoVi4Wh1MmZUqPSecUAMRAdUkXtjk9c2g33xxvVWA7Bc9d4E2hJjzxPoyBWSvYAebNQahxNk3Pio942Gy9kaX79j2evzNLuNvNPpc8ajwBcSbaufWMe3FYfbX8tVsdUA9AhYem36a2U3SNZQtxVF4pGBp9pz327w9debh4hf8jzSHoffh4oxGrC4peAEeJFjdR6rnSPFdzvF1o6gWF1EFqdz1ksWCaYyAgapk2WNPU9i9JK66XqW9xyaikrojcX7oZLnMCKi51k1mr7AefJzjh4iUoJ5SHn",
    "power": 200,
    "sn": "serial number here c1",
    "bus_id": "bus info c1",
    "p2key": "p2keys1 c1"
    },
    {
    "miner_id": "miner2",
    "public_key": "rsa2048:2TuPVgMCHJy5atawrsADEzjP7MCVbyyCA89UW6Wvjp9HrBcvkfoM33uFehzP1wtSX7XwvCGhTPQdjjitsZ9zjbLVqLEnuZVYjZmArhJLFJrukpXxo7yVQBn3BH8bZVR5NBWmnRwvSGThyVgKssvQ2m6uLC9PDM4b7VcyJZGoZrrgTU75e851CcxconxvQZ7CZjbpcZ2N3rv7nzWceUKZUfgqoDyEmwsxM44LeB6Z3Rhfe2gHyZSm2JZj2HeyvyEa23dvehUPvZ8ZpUnsR8mRJThBUWSfPjiX5yX97584h6FEh4W3YAu9AFsmvLQULsYAtmTe6SxWSa8GdBizE5tUW5SfU73cF6Gu1FG8uaNeXCXiR8cEsMPjhmJuwuMYdHsAQbVWzWgsXMgyVFtwep",
    "power": 300,
    "sn": "serial number here c2",
    "bus_id": "bus info c2",
    "p2key": "p2keys1 c2"
    }
]
```

+ challenge.json.sample 激活算力, 发起挑战配置信息

```json
{
    "public_key": "rsa2048:2TuPVgMCHJy5atawrsADEzjP7MCVbyyCA89UW6Wvjp9HrAzK4ZdyEHxs7qVnPrrF3R6w3zNJoHz828bNNJq9f6FPqyq9hsRP47qNXu1mcxeqWmRr8TKTBMLQNzNcjZVm6qX2BiSdXetAZsjPBMC6TyC2smee4s5Mqc4uh5rc5v7Z6nHWGxttHbhHGUCyWtgNGgPevFB2odsTdaXgcgWKtR3zLD6qrbaw631yNEJhverkLMrQJz436L21JWkgXpcTDRYPNWnk7DbztgA6RcgLmve3EG125eW2c2Bj7DkkWAVeWHZnXboDM8kYhAEbfRqUuKwn9K1m9adMqfig4xmM5wxGGABu5dD1gmthQRytLF1y3o2kpTtgrsNyBVTkqV7eMR9qJhUxwiU1rXdQKJ",
    "challenge_key": "ed25519:8FhzmFG24qXxJ9BJLHTxwhxYY4yu4NV8YPxtksmC86Nv"
}
```

+ rsa_signer_key.json 发起挑战用的是rsa2048 keys签名文件

```json
{
  "account_id": "miner-test1",
  "public_key": "rsa2048:2TuPVgMCHJy5atawrsADEzjP7MCVbyyCA89UW6Wvjp9HrBWvq7Yc23gLm9CXF1USvSSR13bqqCKpu5cmWrvC7Ebn5uETJnePKEyG3RhQt8WFayJ1KnsaGa6ZBQDKd9MwFqqwaC6KXpCpTRKWho4NgXVhnCDLRjWehS7gKntWjH42Q1TMRLpWAkMtcAbqEJ2B4HjFiyWFQyHwGyvFYv8SkYKBPLgHy8bSvqJzbwB9C5HLqCbeEurmPV9r7MmBG6BZkpVbyYhJRaurf3RkdpEHJMtvy2WG8YWqbtxjGzkGeu2mC4xj4xGBonYgAVkoXp92TRLg4i3Te8weJnzT74YkzYFdQ7SnjJHPJF156W9o5HX67kqdfFqas6yqxnnVSMkw5o6JQr9iG5mxqy1QLx",
  "private_key": "rsa2048:riiewRJm2wpE3rWTs1ikUc83so8ZXMX8vp9dUTnRgMC8GyfLr99MDwLmsGXjCMrdNrZBdvRZERvZ9HjFTHCqb6cGi4dEV7vcQwJND4Rd4CzHzaYc3WUC25Dmfp6HJM8usxH2mzqfZsvfMgNcPW6n2ESQ44yEkqsetwvn6pAiKgVHEmaY2fa3HEoET8gNjGbpuATRtx5J3nhVf8Wk7v4UFbH5Z234C76rU3bKFXeeLYNDTnqDAjT6LNbRJsdPbMoQfHArmbv3AKzk3QAKwcyq1sxysUtx8uzb4feJ5EjXCR9RyBG27dTwbXfJNruKu49SYCd3Did5ae1bWMGhxzUrXXb7RZq9hikJCmmQLNGX7r5PrSenztiKTqhQxGKpMoHZp7RvReuK8czMidSeJy9EWWXGWrJL5ANN14YguDMpcEf5Db6e3XCJvt3iKfxCjx7pmoSUarqZg4EkXkCMhhkz38f9L6Pedb45RQFZejrWreM5Bh3v1nEWAtUPZY9y4z3jY4ppGP1YBkqLnwc5psZ5bgAVkeRbsWvWA871L2gNXhg5ecPJBQiU8BJDtwEZnyGUBSaSYESQPSwrtzwCxj7FgrxsuJeXamFppgH7C6tisybaaP9M9159ZRFZUjZFwRfWR8ZocxVkHDgBJE7s2hDoMxZxufVxCFeuKzmw7EDmq3AKmLXw7FpUXKwL13NcatuVj2yqiemNchdqwi5Ro51Bp2kv4nGpCWYknRT66KzCGG1DdCJxXpgBpi6q75jx6KeUwpQQKK2AvNm4283M3gaJ4Tnqv4Gzh1tQ5Jt4pK94EYwMSgy62V2bshBRVxTXAzECWkxxmuYtm9ahRMr1mqZi1AfQoswxhAVYvB2g59ZdKs4tSiBtsQZVT7emC5mo7XEFdVqBpcA7wYHKnQ15sevbDyVyt3PsM2Gx4tVKYHUhyA85kUdX8UwGGscXDsEdvdD22QQNSspvBNYBd2XUXv7EimAx11ncxMa9RaCHpizrSV7aSnVHY1GEJD6Y22K1V8jqj54m3h17LRUaHw6Liv2QrqKq5BpyP4rK5SX8H19u76qjZRztkUG4qRX16YhrHTg7KgpMZxpVGCHHce5qKTcKeRHHsyYE1Jzkdp5K8yCFKwtAWUaZeoaqJBWhpxTzBucDrwCzfyVpSNXhkx9W8tmKfW45HNNpD5wAXWoDdJPqJrAMahC2cPthKhkZQfND5WTiiWtRFroVqoAyTbVdMUxvTJUKS1r9CDCFgoJFRPnndz2fbdSSabLMcCasyV7Aq6X2ckVQW4Mq69GJHTyokyWHA7d9nmPQCACdSTtAbnPtdeG4tXmSYqELtCL43hLLx8m2MrDWdFgrpkXjEWdPS8p9QEuv86opKxiU3MjYgBF9oUfdSV4nmbDFR7H1LQ5sKcCxg7Z515VSCVwCKbRgtvtcUV6hmmKu7HBVmyiNEpjy3EWPM2GmPC84qs2wm3j2tamnXgKU9ZtiPpD5KAueYURbxurempUUcBwyLmSiHdfrGiiBZw8FkZRFuAqooa2QxNRzNXEmVGKRVDunUhDi2S4cues452T1tdqpMvtvBwd3EjfT6yNxtSu6o8fjnJqSHRt6JrKQRPagdN7mfV5Rs1zZDv5oaKXUwEwSeyx9pwQoPMPacEkMJ4Ap5pVGdDT8bZn"
}
```

## 各种类型keys生成工具`keypair-generator`

### 生成signer validator node 各种key

```bash
# prepare
git clone https://github.com/utnet-org/utility.git
cd utility

# rsa keys
cargo run --package keypair-generator --bin keypair-generator -- --home=~/keys  --account-id=miner0 --generate-config  signer-keys --key-type=2 --num-keys=4

# signer key
cargo run --package keypair-generator --bin keypair-generator -- --home=~/keys  --account-id=miner0 --generate-config  signer-keys --key-type=0 --num-keys=1

# node key
cargo run --package keypair-generator --bin keypair-generator -- --home=~/keys  --account-id=miner0 --generate-config  node-key

# validator key (a.k.a Create a wallet with keypair-generator)
cargo run --package keypair-generator --bin keypair-generator -- --home=~/keys  --account-id=miner0 --generate-config  validator-key
```

## 至此，节点已经冷启动完毕

可以使用unc/unc-validator相关工具进行账户相关操作, 使用pm2 监控节点
