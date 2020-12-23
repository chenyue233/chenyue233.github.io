# golang高效拼接字符串


### 拼接字符方式

拼接方式均为对同一个长度的字符串重复拼接5万次

```golang
// 声明拼接字符串
const combinString = "golangdafahaogolangdafahaogolangdafahao!!!!"
```



#### 1.  `+`

```golang
func combinStringWithAdd(temp string, n int) string {
	res := ""
	for i := 0; i < n; i++ {
		res += temp
	}
	return res
}
```



#### 2.  `fmt.Sprintf`

```golang
func combinStringWithSprintf(temp string, n int) string {
	res := ""
	for i := 0; i < n; i++ {
		res = fmt.Sprintf("%s%s", res, temp)
	}
	return res
}
```



#### 3. `strings.Builder`

```golang
func combinStringWithbuilder(temp string, n int) string {
	var builder strings.Builder
	for i := 0; i < n; i++ {
		builder.WriteString(temp)
	}
	return builder.String()
}
```



#### 4. `bytes.Buffer`

```golang
func combinStringWithbuffer(temp string, n int) string {
	buf := new(bytes.Buffer)
	for i := 0; i < n; i++ {
		buf.WriteString(temp)
	}
	return buf.String()
}
```



#### 5. `[]byte`

```golang
func combinStringWithbyte(temp string, n int) string {
	buf := make([]byte, 0)
	for i := 0; i < n; i++ {
		buf = append(buf, temp...)
	}
	return string(buf)
}

```



### 性能对比

```
goos: windows
goarch: amd64
pkg: video_sever/combinString/combin
BenchmarkCombinStringWithAdd-8                 1        10247583400 ns/op
BenchmarkCombinStringWithSprintf-8             1        20303203800 ns/op
BenchmarkCombinStringWithbuilder-8           512           2236001 ns/op
BenchmarkCombinStringWithbuffer-8            570           2244424 ns/op
BenchmarkCombinStringWithbyte-8              424           2657066 ns/op
```

从基准测试来看 `+`和`fmt.Sprintf `性能很差，`strings.Builder`和`bytes.Buffer`性能较好。



### 建议 

推荐使用 `strings.Builder` 来拼接字符串。go语言官方也推荐使用`strings.Builder`来拼接字符串

在使用 `strings.Builder`时，如果知道最终结果的字符长度可以通过 `builder.Grow`提前分配内存，来减少程序运行过程中的内存分配次数


