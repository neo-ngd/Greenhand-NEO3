# Neo3 cli 部署调用智能合约

Neo3 上使用 neo-cli 部署和调用智能合约

## 部署启动 neo-cli

首先，需要正确部署并启动 neo-cli，关于 neo-cli 的安装、配置和启动可以参考文档：[https://docs.neo.org/v3/docs/zh-cn/node/cli/setup.html](https://docs.neo.org/v3/docs/zh-cn/node/cli/setup.html)

## 获得 GAS

要在 Neo3 上面部署/调用合约以及发任何交易都需要支付 GAS，所以当 neo-cli 启动好以后我们需要去获取 GAS。

1、首先创建钱包，在 neo-cli 使用 [create wallet](https://docs.neo.org/v3/docs/zh-cn/node/cli/cli.html#create-wallet)  命令创建一个钱包，就会获得一个钱包地址；

2、打开 NEO3 测试币申领页面 [👉](https://neowish.ngd.network/neo3/) ，在 NEO3 地址填写 1 中生成的地址，进行验证后点击领取，等待一个区块的时间(15 秒左右)，就会收到 GAS；

3、在 neo-cli 中使用命令 list asset 查看，就可以看到 2 中申领的 GAS。

## 部署合约

在开发、编译完智能合约之后，使用 neo-cli 的 [deploy](https://docs.neo.org/v3/docs/zh-cn/node/cli/cli.html#deploy) 命令来部署合约。

### 用法

`deploy <nefFilePath> [manifestFile]`

### **参数**

- `nefFilePath` ：NeoVM 的可执行文件 nef 的路径
- `manifestFile` ：可选，manifest.json 文件的路径，manifest 记录了合约的各个接口信息以及配置内容 , 如果为空，则程序会自动匹配与 nef 文件同名的 manifest.json 文件

### **示例**

```jsx
neo> deploy TestContract.nef TestContract.manifest.json  
Script hash: 0xc3361fef35233f9db6354b259be9cde34ba6e5c6
Gas Consumed: 1.203

Signed and relayed transaction with hash=0xab6dd63ea36a7c95580b241f34ba756e62c767813be5d53e02a983f4e561d284
```

合约部署上链，Hash 为 0xc3361fef35233f9db6354b259be9cde34ba6e5c6。

## 调用合约

在部署好合约之后就可以使用 neo-cli 调用了，调用命令是 [invoke](https://docs.neo.org/v3/docs/zh-cn/node/cli/cli.html#invoke)

### 用**法**

`invoke <hash> <operation> [contractParameters=null] [sender=null] [signerAccounts=null]`

### **参数**

- `hash` ：要调用的合约脚本散列
- `operation` ：调用方法名，后面可以输入传入参数
- `contractParameters`: 调用参数，需要传入 JSON 格式的字符串，如果是类型 ByteArray，需要提前进行 Base64 编码

    示例：地址 `NfKA6zAixybBHHpmaPYPDywoqDaKzfMPf9` 可转换为 16 进制大端序的 ScriptHash `0xe4b0b6fa65a399d7233827502b178ece1912cdd4` 也可转换为 Base64 编码的 ScriptHash `1M0SGc6OFytQJzgj15mjZfq2sOQ=` 。JSON 格式的参数如下：

    ```jsx
    [{"type":"ByteArray","value":"1M0SGc6OFytQJzgj15mjZfq2sOQ="}] 
    [{"type":"Hash160","value":"0xe4b0b6fa65a399d7233827502b178ece1912cdd4"}]
    ```

    支持的参数 Type：

    ```jsx
    Any = 0x00,
    Boolean = 0x10,
    Integer = 0x11,
    ByteArray = 0x12,
    String = 0x13,
    Hash160 = 0x14,
    Hash256 = 0x15,
    PublicKey = 0x16,
    Signature = 0x17,
    Array = 0x20,
    Map = 0x22,
    InteropInterface = 0x30,
    Void = 0xff
    ```

- `sender` ：交易发送方，即支付 GAS 费的账户
- `signerAccounts` ：需要添加签名的账户数组，只支持标准账户（单签地址），填写后 neo-cli 会为交易附加该数组内所有地址的签名，这些地址需包含在当前打开的钱包中。

### **示例 1**

示例输入：

```jsx
invoke 0xc3361fef35233f9db6354b259be9cde34ba6e5c6 name
```

示例输出：

```jsx
Invoking script with: '10c00c046e616d650c14f9f81497c3f9b62ba93f73c711d41b1eeff50c2341627d5b52'
VM State: HALT
Gas Consumed: 0.0103609
Evaluation Stack: [{"type":"ByteArray","value":"TXlUb2tlbg=="}]

relay tx(no|yes):no
```

- VM State： `HALT` 表示虚拟机执行成功， `FAULT` 表示虚拟机执行时遇到异常退出。
- Gas Consumed ：调用智能合约时消耗的系统手续费。
- Evaluation Stack：合约执行结果，其中 value 如果是字符串或 ByteArray，则是 Base64 编码后的结果。
- relay tx(no|yes)：是否广播该交易，如果只是查询合约数据，不需要修改链上数据（例如查询 GAS 余额），则输入 no 不广播该交易；如果需要修改链上数据（例如进行转账），则输入 yes 广播该交易。

### **示例 2**

示例输入：

```jsx
invoke 0x230cf5ef1e1bd411c7733fa92bb6f9c39714f8f9 balanceOf [{"type":"ByteArray","value":"1M0SGc6OFytQJzgj15mjZfq2sOQ="}]
```

示例输出：

```jsx
Invoking script with: '0c14d4cd1219ce8e172b50273823d799a365fab6b0e411c00c0962616c616e63654f660c14f9f81497c3f9b62ba93f73c711d41b1eeff50c2341627d5b52'
VM State: HALT
Gas Consumed: 0.0355309
Evaluation Stack: [{"type":"Integer","value":"9999999900000000"}]

relay tx(no|yes): no
```

示例输入：

```jsx
invoke 0x230cf5ef1e1bd411c7733fa92bb6f9c39714f8f9 balanceOf [{"type":"Hash160","value":"0xe4b0b6fa65a399d7233827502b178ece1912cdd4"}]
```

或

```jsx
invoke 0x230cf5ef1e1bd411c7733fa92bb6f9c39714f8f9 balanceOf [{"type":"Hash160","value":"d4cd1219ce8e172b50273823d799a365fab6b0e4"}]
```

示例输出：

```jsx
Invoking script with: '0c14d4cd1219ce8e172b50273823d799a365fab6b0e411c00c0962616c616e63654f660c14f9f81497c3f9b62ba93f73c711d41b1eeff50c2341627d5b52'
VM State: HALT
Gas Consumed: 0.0355309
Evaluation Stack: [{"type":"Integer","value":"9999999900000000"}]

relay tx(no|yes): no
```

### **示例 3**

示例输入：

```jsx
neo> invoke 0xa6a6c15dcdc9b997dac448b6926522d22efeedfb transfer [{"type":"Hash160","value":"0xcb664df6a9852363e19bc9d3c6bb93d91e616a01"},{"type":"Hash160","value":"0xb4ba98beea38621dd96a9804384db24451b1cff2"},{"type":"Integer","value":"1"},{"type":"String","value":"1"}] 0xcb664df6a9852363e19bc9d3c6bb93d91e616a01 NL3TJKJsJi59M49Amub21r1PGF5h2uDKdh
```

示例输出：

```
Invoking script with: 'DAExEQwU8s+xUUSyTTgEmGrZHWI46r6YurQMFAFqYR7Zk7vG08mb4WMjhan2TWbLFMAMCHRyYW5zZmVyDBT77f4u0iJlkrZIxNqXucnNXcGmpkFifVtS'
VM State: HALT
Gas Consumed: 0.099999
Result Stack: [{"type":"Boolean","value":true}]
Relay tx(no|yes): yes
Signed and relayed transaction with hash=0x0e1ad2cc13ff51d3d61081d9f23706ec4d134ddebbd0a8c74054fe34adba345e
```