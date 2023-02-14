# 一筋縄ではいかない　GoでZIPの中のBZIP2を解凍

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/saliton/Go-BZIP2/blob/main/ZIP_BZIP2.ipynb)

大量のテキストファイルを処理する時、何百万ものファイルをファイルシステムに置いたままにしているとなにかと不便なので、zipにまとめてから処理することがあります。そんな時に、zipの圧縮方式をbzip2にした場合の、Go言語で解凍する方法です。一筋縄ではいきませんでした。


まずサンプルを用意します。


```shell
!man man > man.txt
!wc man.txt
```

      724  4977 38134 man.txt


これをzipfileモジュールを使ってZIP_BZIP2でman.zipにアーカイブします。


```python
from zipfile import ZipFile, ZIP_BZIP2
with ZipFile('man.zip', 'w', compression=ZIP_BZIP2, compresslevel=9) as zfile:
    zfile.write('man.txt', 'man.txt')

!zipinfo man.zip
```

    Archive:  man.zip
    Zip file size: 11471 bytes, number of entries: 1
    -rw-r--r--  4.6 unx    38134 b- bzp2 21-Oct-06 08:58 man.txt
    1 file, 38134 bytes uncompressed, 11359 bytes compressed:  70.2%


無事にアーカイブできました。

次にGo言語をインストールします。


```shell
!wget https://golang.org/dl/go1.17.1.linux-amd64.tar.gz
!tar -C /usr/local -xzf go1.17.1.linux-amd64.tar.gz

import os
os.environ['PATH'] += ":/usr/local/go/bin"
```

    --2021-10-06 08:58:27--  https://golang.org/dl/go1.17.1.linux-amd64.tar.gz
    Resolving golang.org (golang.org)... 74.125.142.141, 2607:f8b0:400e:c08::8d
    Connecting to golang.org (golang.org)|74.125.142.141|:443... connected.
    HTTP request sent, awaiting response... 302 Found
    Location: https://dl.google.com/go/go1.17.1.linux-amd64.tar.gz [following]
    --2021-10-06 08:58:27--  https://dl.google.com/go/go1.17.1.linux-amd64.tar.gz
    Resolving dl.google.com (dl.google.com)... 74.125.195.93, 74.125.195.136, 74.125.195.91, ...
    Connecting to dl.google.com (dl.google.com)|74.125.195.93|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 134784143 (129M) [application/x-gzip]
    Saving to: ‘go1.17.1.linux-amd64.tar.gz’
    
    go1.17.1.linux-amd6 100%[===================>] 128.54M   208MB/s    in 0.6s    
    
    2021-10-06 08:58:27 (208 MB/s) - ‘go1.17.1.linux-amd64.tar.gz’ saved [134784143/134784143]
    



```shell
!go version
```

    go version go1.17.1 linux/amd64


それではgo言語でzipの中身を覗いてみましょう。まずは以下でunzip.goファイルにプログラムを書き込みます。


```go
%%writefile unzip.go
package main

import (
    "archive/zip"
    "fmt"
    "log"
)

func main() {
    zfile, _ := zip.OpenReader("man.zip")
    defer zfile.Close()

    for _, f := range zfile.File {
        _, err := f.Open()
        if err != nil {
            fmt.Println(f.Method)
            log.Fatal(err)
        }
        fmt.Println(f.FileInfo().Name())
    }
}
```

    Writing unzip.go


早速実行！


```shell
!go run unzip.go
```

    12
    2021/10/06 08:58:53 zip: unsupported compression algorithm
    exit status 1


そんな圧縮方式知らぬと怒られました。
実はこの時点ですでに何日もかけて大量のテキストファイルをzipにアーカイブした後だったので、やべってなりました。調べた結果、[マニュアル](https://pkg.go.dev/archive/zip#RegisterDecompressor)にこんな記述が！
```
func RegisterDecompressor(method uint16, dcomp Decompressor)
RegisterDecompressor allows custom decompressors for a specified method ID. The common methods Store and Deflate are built in.
```
どうやらカスタム解凍器を設定できるようです。しかし、圧倒的に情報が少ない。method uint16とは何か？
調べたところ、zipアーカイブではファイル毎に圧縮方式を指定できて、圧縮方式毎にIDが決まっているとのこと。ここで[神記事](https://qiita.com/shibukawa/items/67ef687cc28b6a6e56ed)発見。bzip2は12番！

実は後付けですが上記のプログラムでこっそりメソッド番号を出力するコードを忍ばせてありました。ちゃんと12が出力されていますね。

次はdecomp Decompressorですが、これにはcompress/bzip2が使えそう。[神記事](https://qiita.com/shibukawa/items/67ef687cc28b6a6e56ed)によると
> なお、登録する関数はio.ReadCloserを返す必要がありますが、lz4.Readerは単なるio.Readerで、Close()メソッドを持っていません。ioutil.NopCloser()を使うと、ダミーのClose()メソッドを増やしてくれますので使えるようになります。

とのことで、bzip2.Readerも同様に単なるio.Readerのようなので、同様の処置が必要。ただGo1.17ではioutil.NopCloser()ではなくio.NopCloser()を使う必要があるようです。

とうわけで完成したプログラムがこちら



```go
%%writefile unzip_fixed.go
package main

import (
    "archive/zip"
    "compress/bzip2"
    "fmt"
    "io"
    "log"
)

func main() {
    zfile, _ := zip.OpenReader("man.zip")
    defer zfile.Close()

	zfile.RegisterDecompressor(12, func(in io.Reader) io.ReadCloser {
		return io.NopCloser(bzip2.NewReader(in))
	})

    for _, f := range zfile.File {
        rc, err := f.Open()
        if err != nil {
            log.Fatal(err)
        }
        fmt.Println(f.FileInfo().Name())
        if !f.FileInfo().IsDir() {
            buf := make([]byte, f.UncompressedSize)
            n, _ := io.ReadFull(rc, buf)
            fmt.Println(n)
        }
    }
}
```

    Writing unzip_fixed.go



```shell
!go run unzip_fixed.go
```

    man.txt
    38134


ちゃんと解凍できました！

しかし、[神記事](https://qiita.com/shibukawa/items/67ef687cc28b6a6e56ed)がなければ詰むところでした。感謝！
