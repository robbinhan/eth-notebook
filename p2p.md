



节点启动后默认最多查找10个节点

 
每个节点会随机生成一个唯一的NodeID
每个节点只有一个p2p server
会从db中读取上次记录的活跃的节点
每个节点启动的时候会有一个udp服务和一个RLPx服务

各节点之间的通信是通过RLPx协议，RLPx协议是个加密验证的协议，通过tcp方式通信



启动时加上--nodiscover参数时节点不会动态查找其他节点（udp server不会起）
启动时加上--netrestrict参数时不在这个参数范围内的IP都会过滤

datadirStaticNodes : static-nodes.json 启动时就会加载
datadirTrustedNodes : trusted-nodes.json 信任
datadirNodeDatabase : nodes






## p2p/dial.go
```go

// dial performs the actual connection attempt.
// 像远程节点拨号 
func (t *dialTask) dial(srv *Server, dest *discover.Node) error {
	fd, err := srv.Dialer.Dial(dest)
	if err != nil {
		return &dialError{err}
	}
	mfd := newMeteredConn(fd, false)
	return srv.SetupConn(mfd, t.flags, dest)
}

// Dial creates a TCP connection to the node
// srv.Dialer.Dial(dest)实际调用的这里
func (t TCPDialer) Dial(dest *discover.Node) (net.Conn, error) {
	addr := &net.TCPAddr{IP: dest.IP, Port: int(dest.TCP)}
	return t.Dialer.Dial("tcp", addr.String())
}
```

## p2p/server.go
```go
// SetupConn runs the handshakes and attempts to add the connection
// as a peer. It returns when the connection has been added as a peer
// or the handshakes have failed.
func (srv *Server) SetupConn(fd net.Conn, flags connFlag, dialDest *discover.Node) error {
	self := srv.Self()
	if self == nil {
		return errors.New("shutdown")
	}
	c := &conn{fd: fd, transport: srv.newTransport(fd), flags: flags, cont: make(chan error)}
	err := srv.setupConn(c, flags, dialDest)
	if err != nil {
		c.close(err)
		srv.log.Trace("Setting up connection failed", "id", c.id, "err", err)
	}
	return err
}

func (srv *Server) setupConn(c *conn, flags connFlag, dialDest *discover.Node) error {
	// Prevent leftover pending conns from entering the handshake.
	srv.lock.Lock()
	running := srv.running
	srv.lock.Unlock()
	if !running {
		return errServerStopped
	}
	// Run the encryption handshake.
	var err error
	if c.id, err = c.doEncHandshake(srv.PrivateKey, dialDest); err != nil {
		srv.log.Trace("Failed RLPx handshake", "addr", c.fd.RemoteAddr(), "conn", c.flags, "err", err)
		return err
	}
	clog := srv.log.New("id", c.id, "addr", c.fd.RemoteAddr(), "conn", c.flags)
	// For dialed connections, check that the remote public key matches.
	if dialDest != nil && c.id != dialDest.ID {
		clog.Trace("Dialed identity mismatch", "want", c, dialDest.ID)
		return DiscUnexpectedIdentity
	}
	err = srv.checkpoint(c, srv.posthandshake)
	if err != nil {
		clog.Trace("Rejected peer before protocol handshake", "err", err)
		return err
	}
	// Run the protocol handshake
	phs, err := c.doProtoHandshake(srv.ourHandshake)
	if err != nil {
		clog.Trace("Failed proto handshake", "err", err)
		return err
	}
	if phs.ID != c.id {
		clog.Trace("Wrong devp2p handshake identity", "err", phs.ID)
		return DiscUnexpectedIdentity
	}
	c.caps, c.name = phs.Caps, phs.Name
	err = srv.checkpoint(c, srv.addpeer)
	if err != nil {
		clog.Trace("Rejected peer", "err", err)
		return err
	}
	// If the checks completed successfully, runPeer has now been
	// launched by run.
	clog.Trace("connection set up", "inbound", dialDest == nil)
	return nil
}
```


## cmd/utils/flags.go

```go
// 通过geth命令带的参数设置p2p服务的配置
func SetP2PConfig(ctx *cli.Context, cfg *p2p.Config) {
	setNodeKey(ctx, cfg)
	setNAT(ctx, cfg)
	setListenAddress(ctx, cfg)
	setBootstrapNodes(ctx, cfg)
	setBootstrapNodesV5(ctx, cfg)

	if ctx.GlobalIsSet(MaxPeersFlag.Name) {
		cfg.MaxPeers = ctx.GlobalInt(MaxPeersFlag.Name)
	}
	if ctx.GlobalIsSet(MaxPendingPeersFlag.Name) {
		cfg.MaxPendingPeers = ctx.GlobalInt(MaxPendingPeersFlag.Name)
	}
	if ctx.GlobalIsSet(NoDiscoverFlag.Name) || ctx.GlobalBool(LightModeFlag.Name) {
		cfg.NoDiscovery = true
	}

	// if we're running a light client or server, force enable the v5 peer discovery
	// unless it is explicitly disabled with --nodiscover note that explicitly specifying
	// --v5disc overrides --nodiscover, in which case the later only disables v4 discovery
	forceV5Discovery := (ctx.GlobalBool(LightModeFlag.Name) || ctx.GlobalInt(LightServFlag.Name) > 0) && !ctx.GlobalBool(NoDiscoverFlag.Name)
	if ctx.GlobalIsSet(DiscoveryV5Flag.Name) {
		cfg.DiscoveryV5 = ctx.GlobalBool(DiscoveryV5Flag.Name)
	} else if forceV5Discovery {
		cfg.DiscoveryV5 = true
	}

	if netrestrict := ctx.GlobalString(NetrestrictFlag.Name); netrestrict != "" {
		list, err := netutil.ParseNetlist(netrestrict)
		if err != nil {
			Fatalf("Option %q: %v", NetrestrictFlag.Name, err)
		}
		cfg.NetRestrict = list
	}

	if ctx.GlobalBool(DeveloperFlag.Name) {
		// --dev mode can't use p2p networking.
		cfg.MaxPeers = 0
		cfg.ListenAddr = ":0"
		cfg.NoDiscovery = true
		cfg.DiscoveryV5 = false
	}
}
```



## p2p资料
[Peer to Peer](https://github.com/ethereum/go-ethereum/wiki/Peer-to-Peer)

[ÐΞVp2p(libp2p) Wire Protocol](https://github.com/ethereum/wiki/wiki/%C3%90%CE%9EVp2p-Wire-Protocol)

[libp2p白皮书](https://github.com/ethereum/wiki/wiki/libp2p-Whitepaper)

[js的实现](https://github.com/ethereumjs/ethereumjs-devp2p)

[以太坊私網建立](https://medium.com/taipei-ethereum-meetup/%E4%BB%A5%E5%A4%AA%E5%9D%8A%E7%A7%81%E7%B6%B2%E5%BB%BA%E7%AB%8B-%E4%B8%80-43f8677fc9f8)


## RLP(序列化库)

**RLP的唯一目标就是解决结构体的编码问题；对原子数据类型（比如，字符串，整数型，浮点型）的编码则交给更高层的协议；以太坊中要求数字必须是一个大端字节序的、没有零占位的存储的格式（也就是说，一个整数0和一个空数组是等同的）。**

[RLP编码原理](https://github.com/ethereum/wiki/wiki/%5B%E4%B8%AD%E6%96%87%5D-RLP)

[devp2p的rlpx的README](https://github.com/ethereum/devp2p/blob/master/rlpx.md)

