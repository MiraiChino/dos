<!-- page_number: true -->

# Goで始める毎分100万回DoS攻撃

---

# Goってなに？
- Googleがつくったすごい言語
- Python並に読みやすい、C並に速い
![center](img/readability_vs_efficiency.png)
###### 参考：[Why should you learn Go? – Keval Patel – Medium](https://medium.com/@kevalpatel2106/why-should-you-learn-go-f607681fad65)

---

# なにがつくれる？
- Webアプリ
	- YouTubeはGoで書かれている
- ソフトウェア
	- Windows, Linux, Macを同じコードで書ける
	- Android, iOSを同じコードで書ける
- PaaS
	- Docker, KubernetesはGoで書かれている
###### 参考：[go言語（golang）の特徴って？メリットとデメリットをまとめてみた](https://itpropartners.com/blog/10984/)

---

# なんでGo？
<strong style="text-align:center; font-size:130px;">
並列処理が簡単に書ける!!!!!
</strong>


---

# 並列化をやってみたい
- 目標１：６秒の直列処理を３秒でやる
- 目標２：==DoS攻撃==をする
![center](img/dos.png)

###### 参考：[DDoS攻撃とは｜「分かりそう」で「分からない」でも「分かった」気になれるIT用語辞典](https://wa3.i-3-i.info/word11008.html)
###### 参考：[Goルーチンで並行化する方法: 6秒かかる処理を3秒にしよう - Qiita](https://qiita.com/suin/items/82ecb6f63ff4104d4f5d)

---

# ６秒の直列処理を３秒でやる
![center](img/6s.png)

---

# ６秒の直列処理
- 6s.go

	``` go
	func main() {
		log.Print("-- start")

		log.Print("1s")
		time.Sleep(1 * time.Second)
		log.Print("done")

		log.Print("2s")
		time.Sleep(2 * time.Second)
		log.Print("done")

		log.Print("3s")
		time.Sleep(3 * time.Second)
		log.Print("done")

		log.Print("-- finish")
	}
	```

# 3秒の並列処理
- 3s.go

``` go
func main() {
	log.Print("-- start")

	finished := make(chan bool)

	tasks := []func(){
		func() {
			log.Print("1s")
			time.Sleep(1 * time.Second)
			log.Print("done")
			finished <- true
		},
		func() {
			log.Print("2s")
			time.Sleep(2 * time.Second)
			log.Print("done")
			finished <- true
		},
		func() {
			log.Print("3s")
			time.Sleep(3 * time.Second)
			log.Print("done")
			finished <- true
		},
	}

	for _, task := range tasks {
		go task()
	}

	for i := 0; i < len(tasks); i++ {
		<-finished
	}

	log.Print("-- finish")
}
```

- 3skai.go
``` go
func main() {
	log.Print("-- start")

	tasks := []func(){
		sleep(1),
		sleep(2),
		sleep(3),
	}

	var wg sync.WaitGroup

	for _, task := range tasks {
		wg.Add(1)
		go func(f func()) {
			defer wg.Done()
			f()
		}(task)
	}
	wg.Wait()

	log.Print("-- finish")
}

func sleep(sec int) func() {
	return func() {
		log.Printf("%ds", sec)
		time.Sleep(time.Duration(sec) * time.Second)
		log.Printf("%d done", sec)
	}
}

```

# 攻撃されるサーバをローカルに立てる
- server.go

	``` go
	package main

	import (
	    "fmt"
	    "net/http"
	)

	func handler(w http.ResponseWriter, r *http.Request) {
	    fmt.Fprintf(w, "Hello, World")
	}

	func main() {
	http.HandleFunc("/", handler)
	    http.ListenAndServe(":8080", nil)
	}
	```
    
	たったこれだけ！！！

---

# DoS攻撃する
- dos.go

	``` go
	package main

	import (
		"log"
		"net/http"
		"sync"
		"time"
	)

	func main() {
		url := "http://localhost:8080"

		maxConnection := make(chan bool, 100)
		var wg sync.WaitGroup

		count := 0
		start := time.Now()
		for maxRequest := 0; maxRequest < 10000; maxRequest++ {
			wg.Add(1)
			maxConnection <- true
			go func() {
				defer wg.Done()

				resp, err := http.Get(url)
				if err != nil {
					return
				}
				defer resp.Body.Close()

				count++
				<- maxConnection
			}()
		}
		wg.Wait()
		end := time.Now()
		log.Printf("%d 回のリクエストに成功しました！\n", count)
		log.Printf("%f 秒処理に時間がかかりました！\n", (end.Sub(start)).Seconds())
	}
	```
