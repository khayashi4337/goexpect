[![CircleCI](https://circleci.com/gh/google/goexpect.svg?style=svg)](https://circleci.com/gh/google/goexpect)

このパッケージは、[Go](https://golang.org)による、[Expect](https://en.wikipedia.org/wiki/Expect) の実装です。

# forkした動機

sshで、たくさんサーバさらわなくちゃいけなくて、googleの方がおつくりになった、sshのサンプルが、動かないと言う記事をみつけたので、

ここにまとめようと思ったしだいです。


※ 以下、英語が苦手な自分が意訳

## 特徴:
 - 実際のPTYを使用してローカルプロセスを生成および制御します。
 - ネイティブSSHスポーナー
 - テスト用に、調節してあるスポナーになっている.
 - 利用者がスポナーをカスタマイズしやすいように設計してある.
 - そのまま、バッチ実行することもできます。


## オプション変数の与え方

全 Spawn ファンクションは、 expect.Option 型の変数を受け付ける設計になっております。Expecterのオプション変更用です.

### CheckDuration:コマンドプロンプト確認間隔オプション

このGo Expecterは、新しいデータがやってきていないか、２秒間隔で、チェックします。このチェック間隔を変更するには下記のファンクションをつかいます。
```golang
func CheckDuration(d time.Duration) Option
```

### Verbose:詳細ログオプション

このオプションは、主にトラブルシューティング用に、expectや、sendを実行中の、詳細なログ出力のON/OFFを設定します。

### VerboseWriter：ログ出力先指定

このオプションは、上記の冗長オプションのログファイル書き込み先の指定に使います。

```golang
// 下記のExampleVerbose関数は Verbose と VerboseWriter のオプションを変更するサンプルです。
func ExampleVerbose() {
	rIn, wIn := io.Pipe()
	rOut, wOut := io.Pipe()
	waitCh := make(chan error)
	outCh := make(chan string)
	defer close(outCh)

	go fakeCli(cliMap, rIn, wOut)
	go func() {
		var last string
		for s := range outCh {
			if s == last {
				continue
			}
			fmt.Println(s)
			last = s
		}
	}()

	exp, r, err := SpawnGeneric(&GenOptions{
		In:    wIn,
		Out:   rOut,
		Wait:  func() error { return <-waitCh },
		Close: func() error { return wIn.Close() },
		Check: func() bool {
			return true
		}}, -1, Verbose(true), VerboseWriter(os.Stdout))
	if err != nil {
		fmt.Printf("SpawnGeneric failed: %v\n", err)
		return
	}
	re := regexp.MustCompile("testrouter#")
	var interactCmdSorted []string
	for k := range cliMap {
		interactCmdSorted = append(interactCmdSorted, k)
	}
	sort.Strings(interactCmdSorted)
	interact := func() {
		for _, cmd := range interactCmdSorted {
			if err := exp.Send(cmd + "\n"); err != nil {
				fmt.Printf("exp.Send(%q) failed: %v\n", cmd+"\n", err)
				return
			}
			out, _, err := exp.Expect(re, -1)
			if err != nil {
				fmt.Printf("exp.Expect(%v) failed: %v out: %v", re, err, out)
				return
			}
		}
	}
	interact()

	waitCh <- nil
	exp.Close()
	wOut.Close()

	<-r
	// Output:
	// [34mSent:[39m "show system uptime\n"
	// [32mMatch for RE:[39m "testrouter#" found: ["testrouter#"] Buffer: Current time:      1998-10-13 19:45:47 UTC
	// Time Source:       NTP CLOCK
	// System booted:     1998-10-12 20:51:41 UTC (22:54:06 ago)
	// Protocols started: 1998-10-13 19:33:45 UTC (00:12:02 ago)
	// Last configured:   1998-10-13 19:33:45 UTC (00:12:02 ago) by abc
	// 12:45PM  up 22:54, 2 users, load averages: 0.07, 0.02, 0.01
	//
	// testuser@testrouter#
	// [34mSent:[39m "show system users\n"
	// [32mMatch for RE:[39m "testrouter#" found: ["testrouter#"] Buffer: 7:30PM  up 4 days,  2:26, 2 users, load averages: 0.07, 0.02, 0.01
	// USER     TTY FROM              LOGIN@  IDLE WHAT
	// root     d0  -                Fri05PM 4days -csh (csh)
	// blue   p0 level5.company.net 7:30PM     - cli
	//
	// testuser@testrouter#
	// [34mSent:[39m "show version\n"
	// [32mMatch for RE:[39m "testrouter#" found: ["testrouter#"] Buffer: Cisco IOS Software, 3600 Software (C3660-I-M), Version 12.3(4)T
	//
	// TAC Support: http://www.cisco.com/tac
	// Copyright (c) 1986-2003 by Cisco Systems, Inc.
	// Compiled Thu 18-Sep-03 15:37 by ccai
	//
	// ROM: System Bootstrap, Version 12.0(6r)T, RELEASE SOFTWARE (fc1)
	// ROM:
	//
	// C3660-1 uptime is 1 week, 3 days, 6 hours, 41 minutes
	// System returned to ROM by power-on
	// System image file is "slot0:tftpboot/c3660-i-mz.123-4.T"
	//
	// Cisco 3660 (R527x) processor (revision 1.0) with 57344K/8192K bytes of memory.
	// Processor board ID JAB055180FF
	// R527x CPU at 225Mhz, Implementation 40, Rev 10.0, 2048KB L2 Cache
	//
	// 3660 Chassis type: ENTERPRISE
	// 2 FastEthernet interfaces
	// 4 Serial interfaces
	// DRAM configuration is 64 bits wide with parity disabled.
	// 125K bytes of NVRAM.
	// 16384K bytes of processor board System flash (Read/Write)
	//
	// Flash card inserted. Reading filesystem...done.
	// 20480K bytes of processor board PCMCIA Slot0 flash (Read/Write)
	//
	// Configuration register is 0x2102
	//
	// testrouter#

}
```

### NoCheck：ノーチェックオプション

このオプションは、このGo Expecter は、定期的に、スポーンさせた sshや　telnetとかが、使える状態であるかどうか、確認しない様にするオプション

### DebugCheck：　スポーン状況ログオプション

このオプションは、上記のタイミングで、sshやtelnetの状況をログ表示するオプション

### ChangeCheck：スポナーチェックオプション

このオプションは、スポナーチェックする処理を、新しいチェックに置き換えます。

### SendTimeout

Sendコマンドのタイムアウト時刻を指定します。これを指定しない場合は、永遠に待ってることになりますよ。


## 基本的な使用例

### networkbit.ch　というサイトに掲載されていた例
### The [Wikipedia Expect](https://en.wikipedia.org/wiki/Expect) examples.

#### Telnetの例

First we try to replicate the Telnet example from wikipedia as close as possible.

Interaction:

```diff
+ username:
- user\n
+ password:
- pass\n
+ %
- cmd\n
+ %
- exit\n
```

*サンプルをシンプルにするため、エラーチェック指定は省いてあります。*

```golang
package main

import (
	"flag"
	"fmt"
	"log"
	"regexp"
	"time"

	"github.com/google/goexpect"
	"github.com/google/goterm/term"
)

const (
	timeout = 10 * time.Minute
)

var (
	addr = flag.String("address", "", "address of telnet server")
	user = flag.String("user", "", "username to use")
	pass = flag.String("pass", "", "password to use")
	cmd  = flag.String("cmd", "", "command to run")

	userRE   = regexp.MustCompile("username:")
	passRE   = regexp.MustCompile("password:")
	promptRE = regexp.MustCompile("%")
)

func main() {
	flag.Parse()
	fmt.Println(term.Bluef("Telnet 1 example"))

	e, _, err := expect.Spawn(fmt.Sprintf("telnet %s", *addr), -1)
	if err != nil {
		log.Fatal(err)
	}
	defer e.Close()

	e.Expect(userRE, timeout)
	e.Send(*user + "\n")
	e.Expect(passRE, timeout)
	e.Send(*pass + "\n")
	e.Expect(promptRE, timeout)
	e.Send(*cmd + "\n")
	result, _, _ := e.Expect(promptRE, timeout)
	e.Send("exit\n")

	fmt.Println(term.Greenf("%s: result: %s\n", *cmd, result))
}

```

In essence to run and attach to a process the `expect.Spawn(<cmd>,<timeout>)` is used.
The spawn returns and Expecter that can rund `e.Expect` and `e.Send` commands to match information
in the output and Send information in.

*See the https://github.com/google/goexpect/blob/master/examples/newspawner/telnet.go  example for a slightly more fleshed out version*

#### FTPの例

For the FTP example we use the expect.Batch for the following interaction.

```diff
+ username:
- user\n
+ password:
- pass\n
+ ftp>
- prompt\n
+ ftp>
- mget *\n
+ ftp>'
- bye\n
```

*ftp_example.go*

```golang
package main

import (
	"flag"
	"fmt"
	"log"
	"time"

	"github.com/google/goexpect"
	"github.com/google/goterm/term"
)

const (
	timeout = 10 * time.Minute
)

var (
	addr = flag.String("address", "", "address of telnet server")
	user = flag.String("user", "", "username to use")
	pass = flag.String("pass", "", "password to use")
)

func main() {
	flag.Parse()
	fmt.Println(term.Bluef("Ftp 1 example"))

	e, _, err := expect.Spawn(fmt.Sprintf("ftp %s", *addr), -1)
	if err != nil {
		log.Fatal(err)
	}
	defer e.Close()

	e.ExpectBatch([]expect.Batcher{
		&expect.BExp{R: "username:"},
		&expect.BSnd{S: *user + "\n"},
		&expect.BExp{R: "password:"},
		&expect.BSnd{S: *pass + "\n"},
		&expect.BExp{R: "ftp>"},
		&expect.BSnd{S: "bin\n"},
		&expect.BExp{R: "ftp>"},
		&expect.BSnd{S: "prompt\n"},
		&expect.BExp{R: "ftp>"},
		&expect.BSnd{S: "mget *\n"},
		&expect.BExp{R: "ftp>"},
		&expect.BSnd{S: "bye\n"},
	}, timeout)

	fmt.Println(term.Greenf("All done"))
}

```

Using the expect.Batcher makes the standard Send/Expect interactions more compact and simpler to write.





#### 動作しないらしいSSHのサンプル 

With the SSH login example we test out the [expect.Caser](https://github.com/google/goexpect/blob/7f68e6ee0bc89860ff53a5c0d50bcfae61853506/expect.go#L388-L397)
and the [Case Tags](https://github.com/google/goexpect/blob/7f68e6ee0bc89860ff53a5c0d50bcfae61853506/expect.go#L324-L335).

Also for this we'll use the Go Expect native [SSH Spawner](https://github.com/google/goexpect/blob/7f68e6ee0bc89860ff53a5c0d50bcfae61853506/expect.go#L872-L879)
instead of spawning a process.


Interaction:

```diff
+ "Login: "
- user
+ "Password: "
- pass1
+ "Wrong password"
+ "Login"
- user
+ "Password: "
- pass2
+ router#
```

*ssh_example.go*

```golang
package main

import (
	"flag"
	"fmt"
	"log"
	"regexp"
	"time"

	"golang.org/x/crypto/ssh"

	"google.golang.org/grpc/codes"

	"github.com/google/goexpect"
	"github.com/google/goterm/term"
)

const (
	timeout = 10 * time.Minute
)

var (
	addr  = flag.String("address", "", "address of telnet server")
	user  = flag.String("user", "user", "username to use")
	pass1 = flag.String("pass1", "pass1", "password to use")
	pass2 = flag.String("pass2", "pass2", "alternate password to use")
)

func main() {
	flag.Parse()
	fmt.Println(term.Bluef("SSH Example"))

	sshClt, err := ssh.Dial("tcp", *addr, &ssh.ClientConfig{
		User:            *user,
		Auth:            []ssh.AuthMethod{ssh.Password(*pass1)},
		HostKeyCallback: ssh.InsecureIgnoreHostKey(),
	})
	if err != nil {
		log.Fatalf("ssh.Dial(%q) failed: %v", *addr, err)
	}
	defer sshClt.Close()

	e, _, err := expect.SpawnSSH(sshClt, timeout)
	if err != nil {
		log.Fatal(err)
	}
	defer e.Close()

	e.ExpectBatch([]expect.Batcher{
		&expect.BCas{[]expect.Caser{
			&expect.Case{R: regexp.MustCompile(`router#`), T: expect.OK()},
			&expect.Case{R: regexp.MustCompile(`Login: `), S: *user,
				T: expect.Continue(expect.NewStatus(codes.PermissionDenied, "wrong username")), Rt: 3},
			&expect.Case{R: regexp.MustCompile(`Password: `), S: *pass1, T: expect.Next(), Rt: 1},
			&expect.Case{R: regexp.MustCompile(`Password: `), S: *pass2,
				T: expect.Continue(expect.NewStatus(codes.PermissionDenied, "wrong password")), Rt: 1},
		}},
	}, timeout)

	fmt.Println(term.Greenf("All done"))
}
```

### Generic Spawner：オリジナルスポナー

The Go Expect package supports adding new Spawners with the `func SpawnGeneric(opt *GenOptions, timeout time.Duration, opts ...Option) (*GExpect, <-chan error, error)`
function. 

*telnet spawner*

From the [newspawner](https://github.com/google/goexpect/blob/master/examples/newspawner/telnet.go) example.

```golang
func telnetSpawn(addr string, timeout time.Duration, opts ...expect.Option) (expect.Expecter, <-chan error, error) {
	conn, err := telnet.Dial(network, addr)
	if err != nil {
		return nil, nil, err
	}

	resCh := make(chan error)

	return expect.SpawnGeneric(&expect.GenOptions{
		In:  conn,
		Out: conn,
		Wait: func() error {
			return <-resCh
		},
		Close: func() error {
			close(resCh)
			return conn.Close()
		},
		Check: func() bool { return true },
	}, timeout, opts...)
}
```

### Fake Spawnerの例

The Go Expect package includes a Fake Spawner `func SpawnFake(b []Batcher, timeout time.Duration, opt ...Option) (*GExpect, <-chan error, error)`. 
This is expected to be used to simplify testing and faking of interactive workflows.

*Fake Spawner*

```golang
// TestExpect tests the Expect function.
func TestExpect(t *testing.T) {
	tests := []struct {
		name    string
		fail    bool
		srv     []Batcher
		timeout time.Duration
		re      *regexp.Regexp
	}{{
		name: "Match prompt",
		srv: []Batcher{
			&BSnd{`
Pretty please don't hack my chassis

router1> `},
		},
		re:      regexp.MustCompile("router1>"),
		timeout: 2 * time.Second,
	}, {
		name: "Match fail",
		fail: true,
		re:   regexp.MustCompile("router1>"),
		srv: []Batcher{
			&BSnd{`
Welcome

Router42>`},
		},
		timeout: 1 * time.Second,
	}}

	for _, tst := range tests {
		exp, _, err := SpawnFake(tst.srv, tst.timeout)
		if err != nil {
			if !tst.fail {
				t.Errorf("%s: SpawnFake failed: %v", tst.name, err)
			}
			continue
		}
		out, _, err := exp.Expect(tst.re, tst.timeout)
		if got, want := err != nil, tst.fail; got != want {
			t.Errorf("%s: Expect(%q,%v) = %t want: %t , err: %v, out: %q", tst.name, tst.re.String(), tst.timeout, got, want, err, out)
			continue
		}
	}
}
```

*免責事項: このコードはGoogleの公式製品じゃないんだから、動かないとかトラブルとかあっても許してね。（意訳）*
