## 序言

本文的目的是为那些希望理解分片建议详情，乃至去实现它的朋友提供一份相对完整的细节说明和介绍。本文仅作为二次方分片（quadratic sharding）的第一阶段的描述；第二、三、四阶段目前不在讨论范围，同样，超级二次方分片（super-quadratic sharding）*（“Ethereum 3.0”）* 也不在讨论范围。

假设用变量 `c` 来表示一个节点的有效计算能力，那么在一个普通的区块链里，交易容量就被限定为 O(c)，因为每个节点都必须处理所有的交易。二次方分片的目的，就是通过一种双层的设计来增加交易容量。在第一层中是不需要硬分叉的，主链就保持原样。不过，一种被称为**校验器管理和约**（validator manager contract，VMC）的合约需要被发布到主链上，它用来维持分片系统。这个合约中会存在 O(c) 个 **分片** （目前为100），每个分片都像是个独立的“银河”：它具有自己的账户空间，交易需要指定它们自己应该被发布到哪个分片中，并且分片间的通信是受限的（事实上，在第一阶段，不存在这种通信能力）。

分片运行在一个普通的符合最长链规则的权益证明系统中，权益数据将保存在主链上（具体来说，是在VMC中）。所有分片共享一个通用验证器池，这也意味着：任何通过VMC注册的验证器，理论上都可以在任意时间被授权来在任意分片上创建区块。每个分片会有一个 O(c) 的区块大小/气上限（block size/gas limit），这样，系统的整体容量就变成了 O(c^2) 。

分片系统中的大多数用户都会运行两部分程序。(i) 一个在主链上的全节点（需要 O(c) 资源）或轻量节点（需要 O(log(c)) 资源）。 (ii) 一个通过RPC与主链交互的“分片客户端”（由于这个客户端同样运行在当前用户的计算机中，所以它被认为是可信的）；它也可以作为任意分片的轻客户端、作为特定分片的全客户端（用户需要指定他们正在“监视”某个特定的分片），或者作为一个验证器节点。在这些情况下，一个分片客户端的存储和计算需求也将不会超过 O(c)  （除非用户指定他们正在监视 _每个_ 分片；区块浏览器和大型的交易所可能会这么做）。

在本文中， `Collation` （校对块）被用来与 `Block` （区块）相区别，因为： (i) 它们是不同的RLP（Recursive Length Prefix）对象：交易是第0层的对象，collation是用来打包交易的第一层的对象，而block则是用来打包collation（header）的第二层的对象； (ii) 在分片的情景中这更加清晰。通常， `Collation` 必须由 `CollationHeader` （校对块头）和 `TransactionList` （交易列表）组成； `Collation` 的详细格式和 `Witness` （见证人）会在 **无状态客户端** 那节定义。 `Collator` （校对器）是由主链上 **验证器管理合约** 的 `getEligibleProposer` 函数所生成的示例。算法会在随后的章节中介绍。

| Main Chain                                 | Shard Chain            |
|--------------------------------------------|------------------------|
| Block                                      | Collation              |
| BlockHeader                                | CollationHeader        |
| Block Proposer (or `Miner` in PoW chain)   | Collator               |

## 二次方分片（Quadratic Sharding）

### 常量

* `LOOKAHEAD_PERIODS`: 4
* `PERIOD_LENGTH`: 5
* `COLLATION_GASLIMIT`: 10,000,000 gas
* `SHARD_COUNT`: 100
* `SIG_GASLIMIT`: 40000 gas
* `COLLATOR_REWARD`: 0.001 ETH

### 验证器管理合约（Validator Manager Contract，VMC）

我们假定VMC存在于地址 `VALIDATOR_MANAGER_ADDRESS` 上（在已有的“主分片”上），它支持下列函数：

- `deposit(address validationCodeAddr, address returnAddr) returns uint256` ：添加一个验证器到验证器集合中，验证器的大小就是函数调用时的 `msg.value` （比如存入的以太币数量）。这个函数会返回验证器的索引号。 `validationCodeAddr` 用来存储验证代码的地址，这里的“验证代码”指一个单纯的函数，这个函数需要一个32字节的哈希值和一个签名作为输入，如果签名与哈希值匹配则返回1，否则返回0。如果validationCodeAddr这个地址存储的代码不能通过 [单纯性检查合约] 的单纯性验证，deposit函数就会失败。这里的“单纯性验证”包含了对“验证代码”的实际的静态检查以确保它的输出仅仅依赖于它的输入，而不会受任何状态的影响；它的代码没有使用任何会影响状态的操作码（opcode）；并且它的运行也不会导致状态的变动（比如，来防止攻击者创建这样的恶意“验证代码”：当用它校验投票表决时返回true，但当验证器行为不端、甚至已经有行为不端者的证据被提供给这个函数时，它又返回false）。
- `withdraw(uint256 validatorIndex, bytes sig) returns bool` ：校验签名的正确性（例如，一个以0值和 `sha3("withdraw") + sig` 作为数据，附带了200000个气的，对 `validationCodeAddr` 的调用返回1），如果正确，它会将验证器从验证器集合中移除，并退还存入的以太币。
- `getEligibleProposer(uint256 shardId, uint256 period) returns address` ：使用一个区块哈希（block hash）作为种子基于预设的算法从验证器集合中选择一个签名者（signer）。验证器被选中几率应该与其存款数量成正比。这个函数应该可以返回一个当前周期内的值或者以 `LOOKAHEAD_PERIODS` 为上限的任意未来周期内的值。
- `addHeader(bytes header) returns bool` ：尝试处理一个collation header，成功时返回true，失败时返回false。
- `getShardHead(uint256 shardId) returns bytes32` ：返回验证器管理合约内由参数所指定的分片的header哈希。

这里也有一个日志类型：

-   `CollationAdded(indexed uint256 shard_id, bytes collation_header_bytes, bool is_new_head, uint256 score)`

where `collation_header_bytes` can be constructed in vyper by

```python
    collation_header_bytes = concat(
        as_bytes32(shard_id),
        as_bytes32(expected_period_number),
        period_start_prevhash,
        parent_hash,
        transaction_root,
        as_bytes32(collation_coinbase),
        state_root,
        receipt_root,
        as_bytes32(collation_number),
    )
```

Note: `coinbase` and `number` are renamed to `collation_coinbase` and `collation_number`, due to the fact that they are reserved keywords in vyper.

### 校对块头（Collation Header）

我们首先以一个有下列内容的RLP列表来定义一个“collation header”：

    [
        shard_id: uint256,
        expected_period_number: uint256,
        period_start_prevhash: bytes32,
        parent_hash: bytes32,
        transaction_root: bytes32,
        coinbase: address,
        state_root: bytes32,
        receipt_root: bytes32,
        number: uint256,
    ]

这里：

- `shard_id` 分片的ID；
- `expected_period_number` 是collation希望被包含进的周期序号，这是由 `period_number = floor(block.number / PERIOD_LENGTH)` 计算出来的；
- `period_start_prevhash` 前一区块，即区块 `PERIOD_LENGTH * expected_period_number - 1` 的区块哈希（这其实就是希望被包含进的周期起始区块之前的最后一个区块的哈希）。分片中使用区块数据的操作码（例如NUMBER和DIFFICULTY）会使用这个区块的数据，除了COINBASE操作码，它会使用分片的coinbase；
- `parent_collation_hash` 是父collation的哈希；
- `tx_list_root` 是包含在当前collation中的交易数据的查找树（trie）根哈希；
- `post_state_root` 是分片中当前collation之后的新状态根；
- `receipts_root` 是收据查找树（receipt trie）根哈希；
- `sig` 是一个签名。

当 `addHeader(header)` 的调用返回true时， **collation header** 有效。验证器管理合约会在满足下列条件时这么做：

- `shard_id` 是0到 `SHARD_COUNT` 之间的数值；
- `expected_period_number` 与当前周期号相等（比如  `floor(block.number / PERIOD_LENGTH)` ）
- 一个具有相同的分片 `parent_collation_hash` 的collation已经被接受；并且
- `sig` 是一个有效的签名。就是说，如果我们计算 `validation_code_addr = getEligibleProposer(shard_id, current_period)` ，然后使用 `sha3(shortened_header) ++ sig` （这里的 `shortened_header`  是“collation header” _去掉_ sig之后的RLP编码格式）来调用 `validation_code_addr` 的话，调用结果应该为1。

当满足以下条件时， **collation** 有效： (i) 它的“collation header”有效； (ii)  在 `parent_collation_hash` 的 `post_state_root` 上执行collation的结果为给定的 `post_state_root` 和 `receipts_root` ；并且 (iii) 总共使用的气（gas）小于等于 `COLLATION_GASLIMIT` 。

### Collation状态转换函数

执行一个collation时的状态转换处理如下：

* 按顺序执行由 `tx_list_root` 所指定的树上的每个交易；并且
* 将 `COLLATOR_REWARD` 的奖励分配给coinbase。

### `getEligibleProposer` 的细节

这里是用Viper写的一个简单实现：

```python
def getEligibleProposer(shardId: num, period: num) -> address:
    assert period >= LOOKAHEAD_PERIODS
    assert (period - LOOKAHEAD_PERIODS) * PERIOD_LENGTH < block.number
    assert self.num_validators > 0

    h = as_num256(
        sha3(
            concat(
                blockhash((period - LOOKAHEAD_PERIODS) * PERIOD_LENGTH),
                as_bytes32(shardId)
            )
        )
    )
    return self.validators[
        as_num128(
            num256_mod(
                h,
                as_num256(self.num_validators)
            )
        )
    ].addr
```

## 无状态客户端（Stateless Clients）

当验证器被要求在一个给定的分片上创建区块时，一个验证器仅会被给予数分钟的通知（准确地说，就是持续 `LOOKAHEAD_PERIODS * PERIOD_LENGTH` 个区块的通知）。在Ethereum 1.0中，创建一个区块需要为验证交易而访问全部的状态。这里，我们的目标是避免需要验证器保留整个系统的状态（因为这样就将使运算资源需求变为 O(c^2) 了）。取而代之，我们允许验证器在仅知晓根状态（state root）的情况下创建collation，而将其他责任交给交易发送者，由他们提供“见证数据”（witness data），例如Merkle分支，以此来验证交易对账户产生影响的“前状态”（pre-state），并提供足够的信息来计算交易执行后的“后状态根”（post-state root）。

（应该注意到，使用非无状态范式（non-stateless paradigm）来实现分片，理论上是可能的；然而，这需要： (i) 租用存储空间来保持存储的有界性；并且 (ii) 验证器需要使用 O(c) 的时间在一个分片中创建区块。上述方案避免了对这些牺牲的需求。）

### 数据格式

我们修改了交易的格式，以使交易必须指定一个 **访问列表** 来列举出它可以访问的状态（后边我们会更精确的描述这点，这里不妨把它想象为是一个地址列表）。任何在VM执行过程中试图读写交易所指定的访问列表以外的状态，都会返回一个错误。这可以防止这样的攻击：某人发送了一个消耗5百万气的随机执行，然后试图访问一个交易发送者和collator都没有见证人的随机账户；可以防止collator包含进像这样浪费collator时间的交易。

交易发送者必须指定“见证人”（witness），这在被签名的交易体 _之外_ ，但也被打包进交易。这里的见证人是一个Merkle树节点的RLP编码的列表（RLP-encoded list），它是由交易在其访问列表中所指定的状态的组成部分。这使collator仅使用状态根就可以处理交易。在发布collation的时候，collator也会发送整个collation的见证人。

#### 交易打包格式

```python
    [
        [nonce, acct, data....],    # transaction body (see below for specification)
        [node1, node2, node3....]   # witness
    ]
```

#### Collation格式

```python
    [
        [shard_id, ... , sig],   # header
        [tx1, tx2 ...],          # transaction list
        [node1, node2, node3...] # witness
    ]
```

也请参考 ethresearch 上的帖子 [无状态客户端的概念](https://ethresear.ch/t/the-stateless-client-concept/172) 。

### 无状态客户端状态转换函数

通常，我们可以将传统的“有状态”客户端执行状态转换的函数描述为：  `stf(state, tx) -> state'` （或 `stf(state, block) -> state'` ）。在无状态客户端模型中，节点不保存状态，所以 `apply_transaction` 和 `apply_block` 可以写为：

```python
apply_block(state_obj, witness, block) -> state_obj', reads, writes
```

这里， `state_obj` 是一个数据元组，包含了状态根和其他 O(1) 大小的状态数据（使用的气、receipts、bloom filter等等）； `witness` 就是见证人； `block` 就是区块的余下部分。其返回的输出是：

* 一个新的 `state_obj` 包含了新的状态根和其他变量；
* 从见证人那里读取的对象集合（用于区块创建）；和
* 为了组成新的状态查找树而被创建的一组新的状态对象。

这使得函数是“单纯性的”（pure），仅处理小尺寸对象（small-sized objects）（相反的例子就是现行的以太坊状态数据，现在已经 [数百G字节](https://etherscan.io/chart/chaindatasizefull) ），从而使他们可以方便地在分片中使用。

### 客户端逻辑

一个客户端应该有一个如下格式的配置：

```python
{
    validator_address: "0x..." OR null,
    watching: [list of shard IDs],
    ...
}
```

如果指定了validator地址，那么客户端会在主链上检查这个地址是否是有效的validator。如果是，那么在每次在主链上开始一个新周期时（例如，当  `floor(block.number / PERIOD_LENGTH)` 变化的时候），客户端将为所有分片的周期 `floor(block.number / PERIOD_LENGTH) + LOOKAHEAD_PERIODS` 调用  `getEligibleProposer` 。如果这个调用返回了某个分片 `i` 的验证器地址，客户端会运行算法 `CREATE_COLLATION(i)` （参考下文）。

对于 `watching` 列表中的每个分片 `i` ，每当一个新collation header出现在主链上，它就会从分片网络中下载完整的collation，并对其进行校验。它将内部保持追踪所有有效的header（这里的有效性是回溯的，例如，一个header如果是有效的，那么他的父header也应该是有效的），并且将head具有最高得分的分片链接受为主分片链，同时从创世（genesis）collation到head的所有collation都是有效的和可用的。注意，这表示主链的重组 *和* 分片链的重组都将影响分片head。

### 逆向匹配候选head

为了实现监视分片的算法和创建collation，我们要做的第一件事就是使用下面的算法来按由高到低的顺序匹配候选head。首先，假设存在一个（非单纯的、有状态的）方法 `getNextLog()` ，它可以取得某个还没有被匹配的给定分片的最新的 `CollationAdded` 日志。这可以通过逆向匹配最新的区块的所有日志来达成，即从head开始，反方向扫描receipt中的每个区块。我们定义一个有状态的方法 `fetch_candidate_head` ：

```python
unchecked_logs = []
current_checking_score = None

def fetch_candidate_head():
    # Try to return a log that has the score that we are checking for,
    # checking in order of oldest to most recent.
    for i in range(len(unchecked_logs)-1, -1, -1):
        if unchecked_logs[i].score == current_checking_score:
            return unchecked_logs.pop(i)
    # If no further recorded but unchecked logs exist, go to the next
    # isNewHead = true log
    while 1:
        unchecked_logs.append(getNextLog())
        if unchecked_logs[-1].isNewHead is True:
            break
    o = unchecked_logs.pop()
    current_checking_score = o.score
    return o
```

用普通的语言重新表述，这里就是反向扫描 `CollationAdded` 日志（对正确的分片），直到获得一个 `isNewHead = True` 的日志。首先返回那个日志，然后用从老到新的顺序返回所有与那个日志分值相同的、 `isNewHead = False` 的所有最新日志。随后到前一个 `isNewHead = True` 的日志（即确保分值会比前一个NewHead低，但比其他人高），再到这个日志之后的所有具有该分值的最新collation，而后到第四个。

这就是说这个算法确保了首先按照分值的由高到低、然后按照从老到新的顺序检查潜在的候选head。

例如，假定 `CollationAdded` 日志具有以下哈希和分值：

    ... 10 11 12 11 13   14 15 11 12 13   14 12 13 14 15   16 17 18 19 16

然后， `isNewHead` 将被按如下赋值：

    ... T  T  T  F  T    T  T  F  F  F    F  F  F  F  F    T  T  T  T  F

如果我们将collation命名为 A1..A5、 B1..B5、 C1..C5 和 D1..D5 ，那么精确的返回顺序将是：

    D4 D3 D2 D1 D5 B2 C5 B1 C1 C4 A5 B5 C3 A3 B4 C2 A2 A4 B3 A1

### 监视一个分片

如果一个客户端在监视一个分片，它应该去尝试下载和校验那个分片中的所有collation（检查任何给定的collation，仅当其父collation已经被校验过）。要取得head，需要持续调用 `fetch_candidate_head()` ，直到它返回一个被校验过的collation，也就是head。通常情况下它会立即返回一个有效的collation，或者最多因为网络延迟或小规模的攻击导致生成过几个无效或者不可用的collation，而需要稍微尝试几次。只有在遭遇一个真正长时间运行的51%攻击时，这个算法会恶化到 O(N) 的时间。

### `CREATE_COLLATION`

这个处理由三部分组成，第一部分可以被叫做 `GUESS_HEAD(shard_id)` ，其示意代码如下：

```python
# Download a single collation and check if it is valid or invalid (memoized)
validity_cache = {}
def memoized_fetch_and_verify_collation(c):
    if c.hash not in validity_cache:
        validity_cache[c.hash] = fetch_and_verify_collation(c)
    return validity_cache[c.hash]


def main(shard_id):
    head = None
    while 1:
        head = fetch_candidate_head(shard_id)
        c = head
        while 1:
            if not memoized_fetch_and_verify_collation(c):
                break
            c = get_parent(c)
```

`fetch_and_verify_collation(c)` 包含了从分片网络取得 `c` 的所有数据（包括见证人信息）并校验它们的处理。上述算法等价于“选取最长有效链，尽可能的检查有效性，如果其数据无效，则转而处理已知的次长链”。这个算法应该仅当校验器执行超时时才会停止，这就是该创建collation的时候了。每个  `fetch_and_verify_collation` 的执行都应该返回一个“写集合”（参考上文的“无状态客户端”那节）。保存所有这些“写集合”，把它们组合在一起，就构成了  `recent_trie_nodes_db` 。

我们现在可以来定义 `UPDATE_WITNESS(tx, recent_trie_nodes_db)` 了。在运行  `GUESS_HEAD` 的过程中，某节点会接收到一些交易。当它要把交易（尝试）包含进collation的时候，这个算法需要先运行交易。假定交易有一个访问列表  `[A1 ... An]` 和一个见证人 `W` ，对于每个 `Ai` 使用当前状态树的根取得 `Ai` 的Merkle分支，使用 `recent_trie_nodes_db` 和 `W` 一起作为数据库。如果原始的 `W` 正确，并且交易不是在客户端做这些检查之前就已经发出的话，那么这个取得Merkle分支的操作总是会成功的。在将交易包含进collation之后，状态变动的“写集合”也应该被添加到 `recent_trie_nodes_db` 中。

下面我们就要来 `CREATE_COLLATION` 了。作为例证，这里是这个方法中可能的、收集交易信息处理的完整示意代码。

```python
# Sort by descending order of gasprice
txpool = sorted(copy(available_transactions), key=-tx.gasprice)
collation = new Collation(...)
while len(txpool) > 0:
    # Remove txs that ask for too much gas
    i = 0
    while i < len(txpool):
        if txpool[i].startgas > GASLIMIT - collation.gasused:
            txpool.pop(i)
        else:
            i += 1
    tx = copy.deepcopy(txpool[0])
    tx.witness = UPDATE_WITNESS(tx.witness, recent_trie_nodes_db)
    # Try to add the transaction, discard if it fails
    success, reads, writes = add_transaction(collation, tx)
    recent_trie_nodes_db = union(recent_trie_nodes_db, writes)
    txpool.pop(0)
```

最后，有一个额外的步骤，最终确定collation（给collator发放奖励，也就是  `COLLATOR_REWARD` 的ETH）。这需要询问网络以获得collator账户的Merkle分支。当得到网络对此的回应之后，发放奖励之后的“后状态根”（post-state root）就可以被计算出来了。Collator就可以用 (header, txs, witness) 的形式打包这个collation了。这里，见证人（witness）就是所有交易的见证人和collator账户的Merkle分支。

## 协议变动

### 交易的格式

交易的格式现在将变为（注意这里包含了 [账户抽象](https://ethresear.ch/t/tradeoffs-in-account-abstraction-proposals/263/20) 和 [读/写列表](https://ethresear.ch/t/account-read-write-lists/285/3) ）：

```
    [
        chain_id,      # 1 on mainnet
        shard_id,      # the shard the transaction goes onto
        target,        # account the tx goes to
        data,          # transaction data
        start_gas,     # starting gas
        gasprice,      # gasprice
        access_list,   # access list (see below for specification)
        code           # initcode of the target (for account creation)
    ]
```

完成交易的处理过程也将变为：

* 校验 `chain_id` 和 `shard_id` 是正确的；
* 从 `target` 账户中减去 `start_gas * gasprice` wei；
* 检查目标 `account` 是否有代码，如果没有，校验 `sha3(code)[12:] == target` ；
* 如果目标账户为空，使用 `code` 作为初始代码，在 `target` 中执行一个合约的创建；否则，跳过这个步骤；
* 执行一个消息，使用：剩余的气作为startgas， `target` 作为地址，0xff...ff 作为发送者，0作为value，以及当前交易的 `data` 作为data；
* 如果上述任何一个执行失败，并且消耗了 <= 200000 的气（例如：  `start_gas - remaining_gas <= 200000` ），那么这个交易是无效的；
* 否则， `remaining_gas * gasprice` 将被退还，已支付的交易费将被添加到一个交易费计数（注意：交易费*不会*被直接加入coinbase余额，而是在区块最终确认时立即添加）。

### 双层查找树重新设计

现存的账户模型将被替换为：在一个单层查找树中收录进所有账户的余额、代码和存储。具体来讲，这个映射为：

* 账户X的余额： `sha3(X) ++ 0x00`
* 账户X的代码： `sha3(X) ++ 0x01`
* 账户X的存储键值K： `sha3(X) ++ 0x02 ++ K`

请参考ethresearch上的帖子 [单层查找树中的双层账户查找树](https://ethresear.ch/t/a-two-layer-account-trie-inside-a-single-layer-trie/210) 。

此外，这个查找树现在有了一个新的二进制查找树设计： https://github.com/ethereum/research/tree/master/trie_research 。

### 访问列表

一个账号的访问列表看起来大概像这样：

    [[address, prefix1, prefix2...], [address, prefix1, prefix2...], ...]

从根本上说，这意味着：“这个交易可以访问这里给定的所有账户的余额和代码，并且账户列表中给出的每个账户的前缀中至少有一个是该账户存储的一个键的前缀”。我们可以将其转换为“前缀列表格式”，基本上就是一个内部存储查找树的前缀列表（参考前面的章节）：

```python
def to_prefix_list_form(access_list):
    o = []
    for obj in access_list:
        addr, storage_prefixes = obj[0], obj[1:]
        o.append(sha3(addr) + b'\x00')
        o.append(sha3(addr) + b'\x01')
        for prefix in storage_prefixes:
            o.append(sha3(addr) + b'\x02' + prefix)
    return o
```

我们可以通过取得交易的访问列表，将其变换为前缀列表格式，然后对前缀列表中的每个前缀执行 `get_witness_for_prefix` ，并将这些调用结果组成一个集合；以此来计算某个交易见证人。

`get_witness_for_prefix` 会返回查找树节点中可以访问以指定前缀开始的所有键值的一个最小集合。参考这里的实现： https://github.com/ethereum/research/blob/b0de8d352f6236c9fa2244fed871546fabb016d1/trie_research/new_bintrie.py#L250 。

在 EVM 中，任何尝试对访问列表以外的账户的访问（直接调用、SLOAD或者通过类似 `BALANCE` 或 `EXTCODECOPY` 的opcode的操作）都会导致运行这种代码的EVM实例抛出异常。

请参考 ethresearch 上的帖子 [账户读/写列表](https://ethresear.ch/t/account-read-write-lists/285) 。

### Gas 的消耗

待定。

## 后续的阶段

通过分离区块proposer和collator，我们实现了二次方扩展，这是一种快速、不彻底的中等安全权益证明分片，以此在不对协议或软件架构做太多更改的情况下增加了大约100倍的吞吐量。这也被用来作为一个完整的二次方分片多阶段计划的第一阶段，后续阶段大致如下：

* **第二阶段（two-way pegging，即双向限定）** ：参考 `USED_RECEIPT_STORE` 章节，仍在撰写；
* **第三阶段，选项a** ：将collation header作为uncle加入，而不是交易；
* **第三阶段，选项b** ：将collation header加入一个数组，数组中的元素 `i` 必须为分片 `i` 的collation header或者空字符串，并且额外的数据必须为这个数组的哈希（软分叉）；
* **第四阶段（tight coupling，即紧耦合）** ：如果区块指向无效或不可用的collation，那么区块也将变为无效；增加数据可用性证明。

---------------------------------

#### 翻译后记

本文最初是我应以太坊中文社区（Ethfans.org）之邀做的翻译稿，译文于 2018/1/14 首发于【以太坊爱好者】公众号，文章链接：https://mp.weixin.qq.com/s/-VzAL5aR9YFlXHkBG9kkCw。其对应的英文原文最后更新于 2018/1/5，github commit 版本：0d0c74d41dec9ca55d1ff077400229ad524ce10a。

以上正文是我根据最新版原文修订之后的版本，对应的 github commit 版本：8a8fbe298e0490e3acbe20f496fb2aeba59b8a41，更新时间 2018/5/16。

