# IMAP COMPRESS 升级竞态修复（手工修改说明）

目标：修复 `COMPRESS DEFLATE` 后客户端读流仍按明文解析导致的解析错误。

你当前现象是：

- `compress.conn.Write/Flush` 命中
- `compress.conn.Read` 不命中
- 随后出现 `expected a space`、`atom contains forbidden char` 等 parse error

这通常是升级时序竞态：写侧切换成功，但读侧未按 `STARTTLS` 的同步方式完成切换。

---

## 方案 A（推荐）：在 `go-imap` 暴露“可复用的安全升级流程”

`go-imap-compress` 直接调用该流程，避免自行拼接 `Upgrade + Execute` 导致竞态。

---

## 一、需要手工修改的仓库/模块

- `github.com/littleboss01/go-imap`（你本地使用的同一版本）
- `github.com/littleboss01/go-imap-compress`

> 重点：`go-imap` 必须先改，再让 `go-imap-compress` 适配调用。

---

## 二、在 `go-imap` 中改动点（先做）

### 1) 文件：`go-imap/client/client.go`

新增一个**导出方法**（供扩展库调用），封装内部安全升级流程。

方法职责：

- 设置 `c.upgrading = true`
- 在升级回调中先执行“升级触发命令”（例如 COMPRESS 命令）并确认状态成功
- 调用 `c.conn.WaitReady()` 等待读协程进入可升级状态
- 再返回升级后的 `net.Conn`
- 通过 `c.Upgrade(...)` 完成 `Conn` 替换与 `init()`

注意点：

- 不要改变现有 `StartTLS` 逻辑；新方法应复用其时序语义
- 新方法需要能让扩展库传入：
  - 一个“执行升级触发动作”的函数（返回 error）
  - 一个“把旧 conn 包装成新 conn”的函数（返回 `net.Conn, error`）

---

### 2) 文件：`go-imap/client/cmd_noauth.go`

让 `StartTLS` 使用你新增的导出升级方法（可选但建议），确保主库内部和扩展库调用同一路径，降低后续维护差异。

验证目标：

- `StartTLS` 行为保持不变
- 现有测试通过

---

### 3) 文件：`go-imap/conn.go`（通常不需要改）

只做确认，不建议改动：

- `Conn.Upgrade` 内 `c.Conn = upgraded` 与 `c.init()` 保持原样
- `init()` 的 `c.br.Reset(r)` / `c.bw.Reset(w)` 逻辑保持原样

---

## 三、在 `go-imap-compress` 中改动点（后做）

### 文件：`go-imap-compress/client.go`

将当前 `Client.Compress` 中直接使用：

- `c.client.Upgrade(...)`
- 回调里 `c.client.Execute(cmd, nil)`

替换为调用你在 `go-imap` 新增的“安全升级导出方法”。

调用流程应表达为：

- 升级触发动作：执行 `COMPRESS DEFLATE` 命令并检查 `status.Err()`
- 连接包装动作：`createDeflateConn(conn, flate.DefaultCompression)`

最终效果：

- `COMPRESS` 命令确认成功后才切换连接
- 且切换前已完成与读协程同步（`WaitReady` 语义）

---

## 四、调试与验收步骤

### 1) 保留临时调试日志（你已加）

- `deflate connection created`
- `deflate reader is active (first decompression read)`

### 2) 复现流程

- 登录成功后执行 `SupportCompress`、`Compress`
- 连续执行一个会触发服务端返回数据的命令（如 `NOOP`、`SELECT`、`FETCH`）

### 3) 通过标准

- 能看到 `deflate connection created`
- 能看到 `deflate reader is active (first decompression read)`
- 不再出现那组 parse error

### 4) 若仍失败，继续核验

- 运行时是否真使用你修改后的 `go-imap` 与 `go-imap-compress`
- `Compress` 执行时是否有并发命令/后台 `IDLE`
- 服务端是否稳定支持 `COMPRESS=DEFLATE`

---

## 五、为何只改 `go-imap-compress` 不稳

`go-imap-compress` 访问不到 `client.Client` 的私有升级状态（`upgrading` 及其内部配套时序），难以 1:1 复刻 `StartTLS` 的安全切换语义。  
因此从根上修复，必须在 `go-imap` 暴露可复用接口，再由扩展库调用。

---

## 六、你手工改完后的最小自检清单

- [ ] `go-imap` 新导出升级方法已加并可被外部包调用
- [ ] `go-imap-compress/client.go` 不再直接拼 `Upgrade + Execute`，而是调用该方法
- [ ] 压缩启用后读路径能进入 `compress.conn.Read`
- [ ] 无 parse error 复发

---

## 七、修改代码行数与修改内容（按模板）

**文件** `go-imap/client/client.go` **行** `新增约 20~35 行（Client 方法区）` 原代码
```go
// 无此导出方法
```
修改为
```go
func (c *Client) UpgradeWithCommand(
	doUpgradeCommand func() error,
	wrapConn func(net.Conn) (net.Conn, error),
) error {
	c.upgrading = true
	defer func() {
		c.upgrading = false
	}()

	return c.Upgrade(func(conn net.Conn) (net.Conn, error) {
		if err := doUpgradeCommand(); err != nil {
			return nil, err
		}

		c.conn.WaitReady()
		return wrapConn(conn)
	})
}
```

**文件** `go-imap/client/cmd_noauth.go` **行** `修改约 3~8 行（StartTLS 方法内）` 原代码
```go
return c.Upgrade(func(conn net.Conn) (net.Conn, error) {
	if status, err := c.Execute(cmd, nil); err != nil {
		return nil, err
	} else if err := status.Err(); err != nil {
		return nil, err
	}
	return tls.Client(conn, c.tlsConfig), nil
})
```
修改为
```go
return c.UpgradeWithCommand(
	func() error {
		status, err := c.Execute(cmd, nil)
		if err != nil {
			return err
		}
		return status.Err()
	},
	func(conn net.Conn) (net.Conn, error) {
		return tls.Client(conn, c.tlsConfig), nil
	},
)
```

**文件** `go-imap-compress/client.go` **行** `修改约 8~15 行（Compress 方法内）` 原代码
```go
err := c.client.Upgrade(func(conn net.Conn) (net.Conn, error) {
	if status, err := c.client.Execute(cmd, nil); err != nil {
		return nil, err
	} else if err := status.Err(); err != nil {
		return nil, err
	}

	return createDeflateConn(conn, flate.DefaultCompression)
})
```
修改为
```go
err := c.client.UpgradeWithCommand(
	func() error {
		status, err := c.client.Execute(cmd, nil)
		if err != nil {
			return err
		}
		return status.Err()
	},
	func(conn net.Conn) (net.Conn, error) {
		return createDeflateConn(conn, flate.DefaultCompression)
	},
)
```

**文件** `go-imap-compress/deflate.go` **行** `新增 1~3 行（仅调试）` 原代码
```go
func (c *conn) Read(b []byte) (int, error) {
	return c.r.Read(b)
}

func createDeflateConn(c net.Conn, level int) (net.Conn, error) {
	r := flate.NewReader(c)
	w, err := flate.NewWriter(c, level)
	if err != nil {
		return nil, err
	}
	return &conn{
		Conn: c,
		r:    r,
		w:    w,
	}, nil
}
```
修改为
```go
func (c *conn) Read(b []byte) (int, error) {
	println("imap/compress: deflate reader read called")
	return c.r.Read(b)
}

func createDeflateConn(c net.Conn, level int) (net.Conn, error) {
	r := flate.NewReader(c)
	w, err := flate.NewWriter(c, level)
	if err != nil {
		return nil, err
	}
	println("imap/compress: deflate connection created")
	return &conn{
		Conn: c,
		r:    r,
		w:    w,
	}, nil
}
```
