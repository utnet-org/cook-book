# 一. 环境准备
我们的节点基于Ubuntu 2004/2204系统, 下面的操作都是在此系统下部署

## 1. 部署机器

部署相关工作

```sh

# miner-mach 机器预装node.js
# 使用 curl 下载 Node.js 18.x 安装脚本
curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -

# 安装 Node.js
sudo apt-get install -y nodejs

# 安装 pm2
npm install -g pm2

# 安装cargo
curl https://sh.rustup.rs -sSf | sh
rustc --version

```

+ 安装常用的系统工具：netstat、iostat、awk、df
+ awk/df一般都是默认安装的，其它工具可使用以下命令安装：

```sh
apt install net-tools
apt install sysstat
```

+ 安装openssl依赖
+ 

```sh

apt install pkg-config libssl-dev libclang-dev -y
apt-get install --assume-yes libudev-dev

+ 如果cargo依赖的源有无法访问的，需要配置国内的源
在当前用户的~/.cargo/config里面增加如下配置(如果是root用户，则是/root/.cargo/config)
```toml

[source.crates-io]
replace-with = 'tuna'

[source.ustc]
registry = "<https://mirrors.ustc.edu.cn/crates.io-index>"

[source.tuna]
registry = "<https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git>"

[http]
multiplexing = false

```

+ 安装unc cli 和 unc-node

```sh
# 安装 unc 工具
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/utnet-org/utility-cli-rs/releases/download/v0.8.2/utility-cli-rs-installer.sh | sh

# 下载unc-node 节点, 如 2204/2004/2310 任意适合包
wget https://github.com/utnet-org/utility/releases/download/v0.7.3/x86_64-ubuntu-2204-unc-node.tar.gz
wget https://github.com/utnet-org/utility/releases/download/v0.7.3/x86_64-ubuntu-2004-unc-node.tar.gz

# 解压得到 unc-node 二进制文件, 放置在/opt/unc-node/, 后面有用
tar -zxvf x86_64-ubuntu-2204-unc-node.tar.gz

# 安装验证者cli 工具
cargo install unc-validator
```

## 2. 集群机器的统一配置

+ 确保libc的版本，可以运行我们rust编译出来的二进制程序
+ 机器上需要正确安装node(可shell正确执行node和npm命令)， version 18
  
# 二. 配置文件的编写

***这个是新矿工节点的关键步骤***  
下面创建新的描述目录，并在里面增加对应的配置文件

```sh

# testnet node init directly use binaries
unc-node --home /opt/unc-node  init --chain-id testnet --download-genesis --download-config

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
$ rclone copy --no-check-certificate --progress --transfers=6  unc_cf:unc/${latest:?} ~/.unc/data

## 使用 unc cli 创建账户/转账/添加key操作
```sh
## create new accounts 用例, 使用自己创建的account id 而不是用例里面的`fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43`
unc account create-account fund-later use-auto-generation save-to-folder ~/.unc-credentials/implicit
## as follows:
## fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43.json

## 通过unc 基金会账户 给新创建的账户`fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43`  100K unc coin
unc tokens unc send-unc fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43 '100000 unc' network-config testnet sign-with-keychain send

## 在 ～/.unc 目录 `touch validator_key.json`, 验证者信息, 从上面的创建的账户~/.unc-credentials/implicit 信息填写即可  (optional)

{
    "account_id": "fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43"
    "public_key":"ed25519:EYM66gAFekoEgLjicPyvZQiFRrbvNsWjVDQQ11fASV3Q",
    "private_key":"ed25519:3NVx4sHxBJciEH2wZoMig8YiMx1Q84Ur2RWTd2GQ7JNfWdyDxwwYrUR6XtJR3YcYeWh9NzVEmsnYe2keB97mVExZ"
}

```

# 四. 正式部署服务和配置

## 1. 部署脚本

+ unc-node.ecosystem.config.js 文件内容如下:
  其中unc-node 上面的下载节点路径

```json
module.exports = {"apps":[{"name":"unc-node","script":"/opt/unc-node/unc-node","env":{"HOME":"/opt/unc-node"},"exec_mode":"fork","watch":"false","autorestart":true,"restart_delay":5000,"cwd":"/opt/unc-node","args":" --home=/opt/unc-node run"}]}
```

## 2. 服务启动

```sh
pm2 start unc-node.ecosystem.config.js
```

通过pm2 list来查看服务的运行状态  
比如在miner-mach1上，pm2 list显式如下

│ id  │ name               │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├─────┼────────────────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 2   │ unc-node         │ default     │ 0.0.299 │ fork    │ 37988    │ 2h     │ 0    │ online    │ 0%       │ 24.7mb   │ root     │ enabled  │

可见这台机器上的unc-node服务都正常启动了，并且运行正常，没有反复重启

通过pm2 logs 确保unc-node服务已经正确启动了,  链同步快照都下载完成或者同步完成, 没有error 级别日志, 正常打印区块高度

## 3. 发起质押

```sh
## 直接发起质押, 100K unc
unc-validator pledging pledge-proposal fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43 ed25519:EYM66gAFekoEgLjicPyvZQiFRrbvNsWjVDQQ11fASV3Q '100000 UNC' network-config testnet sign-with-keychain send

## 查看直接质押金额
unc-validator pledging view-pledge fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43 network-config testnet now

## view tx status 查看tx 状态值 Success才行
## unc transaction view-status EWHzhriCRTbDVd9SH6Vk88hSHqzJ7pipXW6eUhWTBkvS network-config testnet

```

## 注册矿工上链

```sh
## 使用基金会账户unc 注册Rsa2048 keys
unc extensions register-rsa-keys unc use-file ~/.unc/keys/batch_register.json.sample with-init-call network-config custom sign-with-access-key-file ~/.unc/keys/unc.json send
```

## 添加 rsa key 到账户

```sh
unc account add-key c92fa60934dd1a5a444e171168544d30b7a9dd349786412f8a3003bfc1d126b3 grant-full-access use-manually-provided-public-key rsa2048:2TuPVgMCHJy5atawrsADEzjP7MCVbyyCA89UW6Wvjp9HrAzK4ZdyEHxs7qVnPrrF3R6w3zNJoHz828bNNJq9f6FPqyq9hsRP47qNXu1mcxeqWmRr8TKTBMLQNzNcjZVm6qX2BiSdXetAZsjPBMC6TyC2smee4s5Mqc4uh5rc5v7Z6nHWGxttHbhHGUCyWtgNGgPevFB2odsTdaXgcgWKtR3zLD6qrbaw631yNEJhverkLMrQJz436L21JWkgXpcTDRYPNWnk7DbztgA6RcgLmve3EG125eW2c2Bj7DkkWAVeWHZnXboDM8kYhAEbfRqUuKwn9K1m9adMqfig4xmM5wxGGABu5dD1gmthQRytLF1y3o2kpTtgrsNyBVTkqV7eMR9qJhUxwiU1rXdQKJ network-config testnet sign-with-keychain send
```

## 发起链上挑战

```sh
unc extensions create-challenge-rsa fd09e7537ee95fd2e7b78ee0a2b10bb9db4ebe65dc94802ce420c94ebb25bc43 use-file ~/.unc/keys/challenge.json.sample without-init-call network-config custom sign-with-access-key-file ~/.unc/Unc4/signer_key.json send

```

+ batch_register.json.sample

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

+ challenge.json.sample 文件

```json
{
    "public_key": "rsa2048:2TuPVgMCHJy5atawrsADEzjP7MCVbyyCA89UW6Wvjp9HrAzK4ZdyEHxs7qVnPrrF3R6w3zNJoHz828bNNJq9f6FPqyq9hsRP47qNXu1mcxeqWmRr8TKTBMLQNzNcjZVm6qX2BiSdXetAZsjPBMC6TyC2smee4s5Mqc4uh5rc5v7Z6nHWGxttHbhHGUCyWtgNGgPevFB2odsTdaXgcgWKtR3zLD6qrbaw631yNEJhverkLMrQJz436L21JWkgXpcTDRYPNWnk7DbztgA6RcgLmve3EG125eW2c2Bj7DkkWAVeWHZnXboDM8kYhAEbfRqUuKwn9K1m9adMqfig4xmM5wxGGABu5dD1gmthQRytLF1y3o2kpTtgrsNyBVTkqV7eMR9qJhUxwiU1rXdQKJ",
    "challenge_key": "ed25519:8FhzmFG24qXxJ9BJLHTxwhxYY4yu4NV8YPxtksmC86Nv"
}
```

# 至此，节点已经冷启动完毕

可以使用unc/unc-validator相关工具进行账户相关操作