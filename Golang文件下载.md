---
title: Golang文件下载
created: '2023-09-03T04:06:31.382Z'
modified: '2023-09-04T07:59:08.865Z'
---

# Golang文件下载

#### 普通文件下载

```go
package main

import(
  "io"
  "net/http"
  "os"
)

func main(){
  fileUrl := "http://topgoer.com/static/2/9.png"
  if err :=  DownloadFile("9.png", fileUrl); err != nil{
    panic(err)
  }
}

// download file会将url下载到本地文件，它会在下载时写入而不是将整个文件加载到内存中
// io.Copy 是按默认的缓冲区32k循环操作的，不会将内容一次性全写入内存中,这样就能解决大文件的问题
func DownloadFile(filepath string, url string) error {
  resp, err := http.Get(url)
  if err != nil {
    return err
  }
  defer resp.Body.Close()

  out, err := os.Create(filepath)
  if err != nil {
    return err
  }
  defer out.Close()

  _, err := is.Copy(out, resp.Body)
  return err
}
```

#### 大文件下载
```go
package main

import(
  "fmt"
  "io"
  "net/http"
  "os"
  "strings"

  "github.com/dustin/go-humanize"
)

type WriteCounter struct{
  Total uint64
}

func (wc *WriteCounter) Write(p []byte)(int, error){
  n := len(p)
  wc.Total += uint64(n)
  wc.PrintProgress()
  return n, nil
}

func (wc *WriteCounter) PrintProgress(){
  fmt.Printf("\r%s", strings.Repeat(" ", 35))
  fmt.Printf("\rDownloading... %s complete", humanize.Bytes(wc.Total))
}

func main(){
  fmt.Println("Download Started")
  fileUrl := "http://topgoer.com/static/2/9.png"
  err := DownloadFile("9.png", fileUrl)
  if err != nil {
    panic(err)
  }

  fmt.Println("Download Finished")
}

func DownloadFile(filepath string, url string) error {
  out, err := os.Create(filepath + ".tmp")
  if err != nil {
    return err
  }
  defer out.Close()
  resp, err := http.Get(url)
  if err != nil {
    return err
  }
  defer resp.Body.Close()
  counter := &WriteCounter{}
  if _, err := io.Copy(out, io.TeeReader(resp.Body, counter)); err != nil{
    return err
  }
  fmt.Println()
  if err = os.Rename(filepath + ".tmp", filepath); err != nil {
    return err
  }
  return nil
}

```

#### 大文件断点续传

```go
// gin 框架文件下载支持断点续传
func DownloadHandler(c *gin.Context){
  c.File(filepath)
}

// 使用http代理服务下载文件
func DownloadWithProxy() error {
  // 本地保存文件
  f, err := os.OpenFile(filepath, os.O_APPEND|os.O_CREATE|os.O_RDWR, 0655)
  if err != nil {
    return err
  }
  defer f.Close()

  // http代理下载
  dialer, err != proxy.SOCKS5("tcp", "10.0.0.1:9527", nil, nil)
  if err != nil {
    return err
  }

  httpTransport := &http.Transport{
    Dial: dialer.Dial,
  }

  req, err := http.NewRequest(http.MethodPost, "http://127.0.0.1/api/share/file", nil)
  if err != nil {
    return err
  }

  httpClient := &http.Client{
    Transport: httpTransport, 
    Timeout: 5,
  }

  // 下载文件的本地状态
  fileStat, err := f.Stat()
  if err != nil {
    return err
  }

  var bufByte int64 = 1024 * 2000
  // 文件已有的数据作为文件下载起始位置
  startPosition := fileStat.Size()
  endPosition := startPosition + bufByte

  for {
    // log.Info("start=========", startPosition)
    // log.Info("end===========", endPosition)

    // 判断文件是否已经下载完成

    // 下载文件的请求范围
    req.Header.Set("Accept-Ranges", "bytes")
    req.Header.Set("Range", fmt.Sprintf("bytes=%d-%d", startPosition, endPosition))
    resp, err := httpClient.Do(req)
    if err != nil {
      return err
    }
    defer resp.Body.Close()

    // 文件下载请求错误
    if resp.StatusCode != http.StatusPartialContent {
      return errors.New("file request err: %s", resp.Status)
    }

    // 追加方式保存下载数据
    _, err = io.Copy(f, resp.Body)
    if err != nil {
      return err
    }

    // 请求返回头数据解析已经下载的范围
    var startRange, endRange, total int64
    fmt.Sscanf(resp.Header.Get("Content-Range"), "bytes %d-%d/%d", &startRange, &endRange, &total)

    // 文件下载完成
    if endRange == total - 1 {
      // 文件md5校验
      // 标记文件下载完成
      return
    }

    startPosition = endRange + 1
    endPosition = endRange + bufByte
    if startPosition == total {
      return
    }

    if endPositon > total {
      endPositon = total
    }

  }
}
```

#### 文件MD5
```go
package main

import(
  "crypto/md5"
  "log"
  "fmt"
  "io"
  "os"
)

func main(){
  file, err := os.Open("test.txt")
  if err != nil {
    log.Fatal(err)
  }
  defer file.Close()

  hasher := md5.New()
  _, err := io.Copy(hasher, file)
  if err != nil {
    log.Fatal(err)
  }

  sum := hasher.Sum(nil)
  fmt.Printf("MD5 checksum: %x\n", sum)
}
```


