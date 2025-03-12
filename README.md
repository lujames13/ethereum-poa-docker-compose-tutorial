# 使用Docker Compose建立Geth 1.13私有以太坊網絡

本文檔提供使用Docker Compose和Geth 1.13版本搭建私有以太坊網絡的完整步驟，採用Clique PoA (Proof of Authority)共識機制。

## 檔案架構

```
.
├── README.md
├── docker-compose.yml
├── genesis.json
├── node1
│   ├── geth
│   ├── geth.ipc
│   ├── history
│   └── keystore
├── node1_password.txt
├── node2
│   ├── geth
│   ├── geth.ipc
│   └── keystore
└── node2_password.txt
```

## 前置條件

- 已安裝Docker和Docker Compose
- 基本的終端操作知識

## 完整步驟

### 1. 建立工作目錄

```bash
mkdir -p ethereum-private-net/{node1,node2}
cd ethereum-private-net
```

### 2. 建立密碼文件

```bash
# 為節點1創建密碼文件
echo "password1" > node1_password.txt

# 為節點2創建密碼文件
echo "password2" > node2_password.txt
```

### 3. 為節點創建帳戶

```bash
# 為節點1創建賬戶
docker run --rm -v $(pwd)/node1:/root/.ethereum -v $(pwd)/node1_password.txt:/password.txt ethereum/client-go:v1.13.14 account new --password /password.txt

# 為節點2創建賬戶
docker run --rm -v $(pwd)/node2:/root/.ethereum -v $(pwd)/node2_password.txt:/password.txt ethereum/client-go:v1.13.14 account new --password /password.txt
```

記錄每個命令輸出中的賬戶地址，例如：`0x1234567890abcdef1234567890abcdef12345678`
(在這個範例中，節點1地址為`0xC0A55ae58fb8E26f7874E865eE143f033D445927`，節點2地址為`0x8c59707CcF4c996bDB6163A3a759baADf82dAe6A`)

### 4. 創建創世塊配置文件

創建`genesis.json`文件：

```bash
cat > genesis.json << EOF
{
  "config": {
    "chainId": 123454321,
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0,
    "constantinopleBlock": 0,
    "petersburgBlock": 0,
    "istanbulBlock": 0,
    "muirGlacierBlock": 0,
    "berlinBlock": 0,
    "londonBlock": 0,
    "arrowGlacierBlock": 0,
    "grayGlacierBlock": 0,
    "clique": {
      "period": 5,
      "epoch": 30000
    }
  },
  "difficulty": "1",
  "gasLimit": "800000000",
  "extradata": "0x0000000000000000000000000000000000000000000000000000000000000000NODE1_ADDRESS_WITHOUT_0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  "alloc": {
    "NODE1_ADDRESS_WITHOUT_0x": { "balance": "1000000000000000000" },
    "NODE2_ADDRESS_WITHOUT_0x": { "balance": "1000000000000000000" }
  }
}
EOF
```

**重要**：將以下內容替換為實際值：
- `NODE1_ADDRESS_WITHOUT_0x`: 節點1的地址（去掉0x前綴）
  (在這個範例中為`C0A55ae58fb8E26f7874E865eE143f033D445927`)
- `NODE2_ADDRESS_WITHOUT_0x`: 節點2的地址（去掉0x前綴）
  (在這個範例中為`8c59707CcF4c996bDB6163A3a759baADf82dAe6A`)

### 5. 初始化節點

```bash
# 初始化節點1
docker run --rm -v $(pwd)/node1:/root/.ethereum -v $(pwd)/genesis.json:/genesis.json ethereum/client-go:v1.13.14 init --datadir /root/.ethereum /genesis.json

# 初始化節點2
docker run --rm -v $(pwd)/node2:/root/.ethereum -v $(pwd)/genesis.json:/genesis.json ethereum/client-go:v1.13.14 init --datadir /root/.ethereum /genesis.json
```

### 6. 獲取節點1的enode ID

臨時啟動節點1來獲取enode ID：

```bash
docker run --rm -it --name temp-node1 -v $(pwd)/node1:/root/.ethereum -v $(pwd)/node1_password.txt:/password.txt ethereum/client-go:v1.13.14 --datadir /root/.ethereum --port 30306 --networkid 123454321 console
```

在geth控制台中執行：

```javascript
admin.nodeInfo.enode
```

這將輸出類似於 `enode://7e302aa...@127.0.0.1:30306`的字符串。複製`enode://`之後和`@`之前的部分，這就是節點1的enode ID。
(在這個範例中為`enode://67cbdce9f4d0a82cfb76053f948bb467d5acb96175a31330b99df0907e65a0468946f2ba2680a850e72a00ab92bbed53d417765b0e32e7afb3523453399edd45@111.241.136.73:30306`)

輸入`exit`退出控制台。

### 7. 創建Docker Compose配置文件

創建`docker-compose.yml`文件：

```yaml
version: '3'

services:
  node1:
    container_name: ethereum-node1
    image: ethereum/client-go:v1.13.14
    volumes:
      - ./node1:/root/.ethereum
      - ./node1_password.txt:/password.txt
      - ./genesis.json:/genesis.json
    ports:
      - "30306:30306"
      - "30306:30306/udp"
      - "8545:8545"
    command: --datadir /root/.ethereum --port 30306 --networkid 123454321 --unlock 0xC0A55ae58fb8E26f7874E865eE143f033D445927 --password /password.txt --mine --miner.etherbase 0xC0A55ae58fb8E26f7874E865eE143f033D445927 --http --http.api eth,net,web3,personal,admin --http.addr 0.0.0.0 --http.port 8545 --http.corsdomain "*" --allow-insecure-unlock
    networks:
      ethnet:
        ipv4_address: 172.20.0.2

  node2:
    container_name: ethereum-node2
    image: ethereum/client-go:v1.13.14
    depends_on:
      - node1
    volumes:
      - ./node2:/root/.ethereum
      - ./node2_password.txt:/password.txt
      - ./genesis.json:/genesis.json
    ports:
      - "30307:30307"
      - "30307:30307/udp"
      - "8546:8546"
    command: --datadir /root/.ethereum --port 30307 --networkid 123454321 --unlock 0x8c59707CcF4c996bDB6163A3a759baADf82dAe6A --password /password.txt --http --http.api eth,net,web3,personal,admin --http.addr 0.0.0.0 --http.port 8546 --http.corsdomain "*" --allow-insecure-unlock --bootnodes enode://67cbdce9f4d0a82cfb76053f948bb467d5acb96175a31330b99df0907e65a0468946f2ba2680a850e72a00ab92bbed53d417765b0e32e7afb3523453399edd45@172.20.0.2:30306
    networks:
      ethnet:
        ipv4_address: 172.20.0.3

networks:
  ethnet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

將以下內容替換為實際值：
- `0xNODE1_ADDRESS`: 節點1的完整地址（包含0x前綴）
  (在這個範例中為`0xC0A55ae58fb8E26f7874E865eE143f033D445927`)
- `0xNODE2_ADDRESS`: 節點2的完整地址（包含0x前綴）
  (在這個範例中為`0x8c59707CcF4c996bDB6163A3a759baADf82dAe6A`)
- `NODE1_ENODE_WITHOUT_IP`: 節點1的enode ID
  (在這個範例中為`67cbdce9f4d0a82cfb76053f948bb467d5acb96175a31330b99df0907e65a0468946f2ba2680a850e72a00ab92bbed53d417765b0e32e7afb3523453399edd45`)

### 8. 啟動私有網絡

```bash
docker-compose up
```

如果您想在後台運行：

```bash
docker-compose up -d
```

### 9. 連接到節點控制台

在新的終端窗口中：

```bash
# 連接到節點1
docker exec -it ethereum-node1 geth attach /root/.ethereum/geth.ipc

# 連接到節點2
docker exec -it ethereum-node2 geth attach /root/.ethereum/geth.ipc
```

架構關係：

```
[您的終端] → [Docker容器] → [Geth JavaScript控制台] → [以太坊節點] → [區塊鏈網絡]
```

### 10. 驗證網絡連接

連接到節點控制台後，您將看到類似以下的歡迎信息：

```
Welcome to the Geth JavaScript console!
instance: Geth/v1.13.14-stable-2bd6bd01/linux-amd64/go1.21.7
coinbase: 0xc0a55ae58fb8e26f7874e865ee143f033d445927
at block: 9 (Wed Mar 12 2025 21:39:08 GMT+0000 (UTC))
 datadir: /root/.ethereum
 modules: admin:1.0 clique:1.0 debug:1.0 engine:1.0 eth:1.0 miner:1.0 net:1.0 rpc:1.0 txpool:1.0 web3:1.0
To exit, press ctrl-d or type exit
```

在節點控制台中執行以下命令驗證網絡連接：

```javascript
// 檢查區塊高度 - 數字會隨著挖礦而增加
> eth.blockNumber
10  // 表示目前已經產生了10個區塊

// 檢查對等節點數量
> net.peerCount
2  // 表示已連接到2個節點，通常是另一個本地節點和一個外部節點

// 查看對等節點詳情
> admin.peers
[{
    caps: ["eth/68", "snap/1"],
    enode: "enode://aa6c5c109f9cd6c4d2fa95ed61b1a158c14d0da3934ea3fb744ad28b902a1771a97f2f3eb342cb85385e1d8c8ef9390d3fc8fb8fb948aa767a60622c36bf7d55@172.20.0.3:45978",
    id: "4f674866b3b7ca76ca3650ddf81a138ed1bd68aa6b311a04571a613119e06111",
    name: "Geth/v1.13.14-stable-2bd6bd01/linux-amd64/go1.21.7",  // 顯示這是節點2
    network: {
      inbound: true,  // 表示這是一個入站連接
      localAddress: "172.20.0.2:30306",  // 本地節點地址
      remoteAddress: "172.20.0.3:45978",  // 遠端節點地址(節點2)
      static: false,
      trusted: false
    },
    protocols: {
      eth: {
        version: 68  // 以太坊協議版本
      },
      snap: {
        version: 1  // 快照同步協議版本
      }
    }
}, {
    caps: ["eth/66", "eth/67", "eth/68", "snap/1"],
    enode: "enode://8ffcf8ba02dc25d1fd0cafeeecea2f6ee6c9f5c12199178ac61ee548684b728018ba686a4a4112076d3c8b7ec94c3eb58e7cbb4a9d59a03edf61fb65b37f5dfe@51.195.39.168:30303",
    id: "cb5260385409e7741a7d16ec47c24d4da4c1c6b5f979d9d2df4ca29458bb7267",
    name: "Geth/v1.14.13-stable-eb00f169/linux-amd64/go1.23.5",  // 這是一個外部以太坊節點
    network: {
      inbound: false,  // 表示這是一個出站連接
      localAddress: "172.20.0.2:46520",
      remoteAddress: "51.195.39.168:30303",  // 外部節點的地址
      static: false,
      trusted: false
    },
    protocols: {
      eth: "handshake",  // 正在進行握手
      snap: "handshake"
    }
}]
```

成功連接後，您應該能看到節點1和節點2相互連接，上述輸出中清楚顯示節點1(172.20.0.2)已連接到節點2(172.20.0.3)。

### 11. 檢查賬戶和餘額

在任一節點控制台中執行以下命令：

```javascript
// 顯示賬戶地址
> eth.accounts
["0xc0a55ae58fb8e26f7874e865ee143f033d445927"]  // 列出節點的所有賬戶地址

// 檢查賬戶餘額（以Wei為單位）
> eth.getBalance(eth.accounts[0])
1000000000000000000  // 表示餘額為1 ETH(10^18 Wei)

// 將Wei轉換為更易讀的以太單位
> web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
1  // 確認餘額為1 ETH
```

### 12. 測試交易功能

在節點1的控制台中執行以下命令發送交易：

```javascript
// 從節點1向節點2發送0.1個以太幣的交易
> eth.sendTransaction({
  from: "0xC0A55ae58fb8E26f7874E865eE143f033D445927",  // 節點1地址
  to: "0x8c59707CcF4c996bDB6163A3a759baADf82dAe6A",    // 節點2地址
  value: web3.toWei(0.1, "ether")
})
"0x58b6458922733da50d6230560cd033d147532beb64107e75dc246853cdb6a8ec"  // 返回交易哈希
```

使用返回的交易哈希檢查交易詳情：

```javascript
// 查詢交易詳情
> eth.getTransaction("0x58b6458922733da50d6230560cd033d147532beb64107e75dc246853cdb6a8ec")
{
  accessList: [],
  blockHash: "0x669ca015346779881a8cf844dba3038211e68842bc0b017c584ad137f448bcaa",  // 包含此交易的區塊哈希
  blockNumber: 72,  // 交易被包含在第72個區塊中
  chainId: "0x75bc371",  // 鏈ID，對應十進制123454321
  from: "0xc0a55ae58fb8e26f7874e865ee143f033d445927",  // 發送方地址(節點1)
  gas: 21000,  // 使用的燃料限制
  gasPrice: 1000066773,  // 燃料價格
  hash: "0x58b6458922733da50d6230560cd033d147532beb64107e75dc246853cdb6a8ec",  // 交易哈希
  input: "0x",  // 交易輸入數據(空表示純轉賬交易)
  maxFeePerGas: 1000174426,  // EIP-1559交易最大費用
  maxPriorityFeePerGas: 1000000000,  // EIP-1559交易最大優先費用
  nonce: 0,  // 發送方的交易計數
  r: "0x1d2bb3ce66485791e1de1ecbaa2d06b7eb643b21a425c16bd690d011971c3d54",  // 簽名組件r
  s: "0x28209c4bf004642f4a12863093a1f0e4a8dca71995e0775bf5e20a43bc5c1f1f",  // 簽名組件s
  to: "0x8c59707ccf4c996bdb6163a3a759baadf82dae6a",  // 接收方地址(節點2)
  transactionIndex: 0,  // 交易在區塊中的索引
  type: "0x2",  // 交易類型(0x2表示EIP-1559交易)
  v: "0x1",  // 簽名組件v
  value: 100000000000000000,  // 交易金額(0.1 ETH)
  yParity: "0x1"  // 簽名的y-parity
}
```

在節點2的控制台中檢查餘額變化：

```javascript
web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
```

### 13. 監控區塊生成

在任一節點控制台中，設置一個區塊監控器來實時觀察新區塊生成：

```javascript
// 設置區塊監控器
> eth.filter("latest").watch(function() {
  console.log("新區塊: ", eth.blockNumber);
})
{
  callbacks: [function()],
  filterId: "0x69b1cfd917f61250c96e114bd68cfcef",  // 過濾器ID，用於識別此監控器
  getLogsCallbacks: [],
  implementation: {
    getLogs: function(),
    newFilter: function(),
    poll: function(),
    uninstallFilter: function()
  },
  options: "latest",  // 監控最新區塊
  pollFilters: [],
  requestManager: {
    polls: {
      0x69b1cfd917f61250c96e114bd68cfcef: {
        data: {...},
        id: "0x69b1cfd917f61250c96e114bd68cfcef",
        callback: function(error, messages),
        uninstall: function bound()
      }
    },
    provider: {
      send: function github.com/ethereum/go-ethereum/internal/jsre.MakeCallback.func1(),
      sendAsync: function github.com/ethereum/go-ethereum/internal/jsre.MakeCallback.func1()
    },
    timeout: {},
    poll: function(),
    reset: function(keepIsSyncing),
    send: function(data),
    sendAsync: function(data, callback),
    sendBatch: function(data, callback),
    setProvider: function(p),
    startPolling: function(data, pollId, callback, uninstall),
    stopPolling: function(pollId)
  },
  formatter: function(log),
  get: function(callback),
  stopWatching: function(callback),  // 可用於停止監控
  watch: function(callback)
}

// 隨著新區塊生成，控制台會輸出區塊高度
> 新區塊:  768
新區塊:  769
新區塊:  770
新區塊:  771
新區塊:  772
新區塊:  773
新區塊:  774
新區塊:  775
```

可以看到區塊正在持續生成，表示挖礦功能正常運作。每5秒左右（根據配置的Clique PoA共識週期）就會生成一個新的區塊。

確認餘額變化（收到交易後，節點2的餘額應增加）：

```javascript
// 在節點2控制台中查看餘額
> web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
1.1  // 顯示已收到0.1 ETH，總餘額現為1.1 ETH
```

### 14. 停止網絡

要停止網絡：

```bash
# 如果在前台運行，按Ctrl+C
# 如果在後台運行
docker-compose down
```

## 故障排除

### 節點不能相互連接

1. 檢查網絡ID是否匹配（範例中使用123454321）
2. 確認enode URL是否正確
3. 嘗試重啟容器
4. 檢查Docker日誌尋找錯誤信息

```bash
docker logs ethereum-node1
docker logs ethereum-node2
```

### 無法挖礦

確認節點1的賬戶已解鎖且配置為挖礦者：

```javascript
// 在節點1控制台中
eth.coinbase
miner.getHashrate()
```

### 常見錯誤：無效IP地址

如果遇到 `Fatal: Option nat: invalid IP address` 錯誤，請移除 `--nat extip:node1` 參數並重新啟動。

## 進階用途

此私有網絡可用於：
- 智能合約開發和測試
- DApp開發
- 區塊鏈概念學習與實驗
- 性能測試

### 智能合約部署示例

在節點控制台中：

```javascript
// 解鎖賬戶（如果尚未解鎖）
personal.unlockAccount(eth.accounts[0], "password1", 3600)

// 部署合約代碼示例
var contractSource = `
pragma solidity ^0.8.0;

contract SimpleStorage {
    uint256 private storedData;
    
    function set(uint256 x) public {
        storedData = x;
    }
    
    function get() public view returns (uint256) {
        return storedData;
    }
}
`

// 編譯和部署步驟會因開發環境而異
```

## 注意事項

- 此設置僅用於開發和測試，不適合生產環境
- 密碼文件應妥善保存，生產環境中應使用更安全的方式管理密鑰
- 預設配置的節點暴露了RPC端口，生產環境中應設置適當的安全措施
- 在實際應用中，建議使用更多節點以提高網絡彈性

## 參考資源

- [Geth官方文檔](https://geth.ethereum.org/docs/)
- [以太坊開發者文檔](https://ethereum.org/developers/)
- [Clique PoA共識算法](https://eips.ethereum.org/EIPS/eip-225)