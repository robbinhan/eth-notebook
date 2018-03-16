## 以太坊TPS 优化

1. core/tx_pool.go 的add 方法中`pool.journalTx`的调用。每提交一次交易请求都会将交易信息写一次`transactions.rlp`文件，节点重启时会重新load 文件中的交易。
2. 每次提交交易请求，如果没有传递nonce 参数，会对账号加锁（压力测试需要注意），避免统一账号的nonce 值冲突。（nonce 是每个账号的交易数的counter）
3. 由于第2 项nonce 每次会递增，所以每次调用交易请求后都会执行一次promoteExecutables 方法。如果一次并发很多交易，大部分执行promoteExecutables 是重复操作，较浪费资源。