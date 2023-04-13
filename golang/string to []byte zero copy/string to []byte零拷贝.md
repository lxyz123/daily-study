## 源版
```go
func StringToBytes(s string) (b []byte) {
	sh := *(*reflect.StringHeader)(unsafe.Pointer(&s))
	bh := *(*reflect.SliceHeader)(unsafe.Pointer(&b))
	bh.Data, bh.Len, bh.Cap = sh.Data, sh.Len, sh.Len
	return b
}

// BytesToString converts byte slice to string without a memory allocation.
func BytesToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```
## 改进版
```go
package bytesconv

import (
	"unsafe"
)

// StringToBytes converts string to byte slice without a memory allocation.
func StringToBytes(s string) []byte {
	return *(*[]byte)(unsafe.Pointer(
		&struct {
			string
			Cap int
		}{s, len(s)},
	))
}

// BytesToString converts byte slice to string without a memory allocation.
func BytesToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```
## 通用版本
```go
func GeneralStringToBytes(s string) []byte {
	return []byte(s)
}

func GeneralBytesToString(b []byte) (s string) {
	return string(b)
}
```
## 源码
```go
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte

func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{}
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s))
	}
	copy(b, s)
	return b
}

// rawbyteslice allocates a new byte slice. The byte slice is not zeroed.
func rawbyteslice(size int) (b []byte) {
	cap := roundupsize(uintptr(size))
	p := mallocgc(cap, nil, false)
	if cap != uintptr(size) {
		memclrNoHeapPointers(add(p, uintptr(size)), cap-uintptr(size))
	}

	*(*slice)(unsafe.Pointer(&b)) = slice{p, size, int(cap)}
	return
}

// slicebytetostring converts a byte slice to a string.
// It is inserted by the compiler into generated code.
// ptr is a pointer to the first element of the slice;
// n is the length of the slice.
// Buf is a fixed-size buffer for the result,
// it is not nil if the result does not escape.
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) (str string) {
	if n == 0 {
		// Turns out to be a relatively common case.
		// Consider that you want to parse out data between parens in "foo()bar",
		// you find the indices and convert the subslice to string.
		return ""
	}
	if raceenabled {
		racereadrangepc(unsafe.Pointer(ptr),
			uintptr(n),
			getcallerpc(),
			abi.FuncPCABIInternal(slicebytetostring))
	}
	if msanenabled {
		msanread(unsafe.Pointer(ptr), uintptr(n))
	}
	if asanenabled {
		asanread(unsafe.Pointer(ptr), uintptr(n))
	}
	if n == 1 {
		p := unsafe.Pointer(&staticuint64s[*ptr])
		if goarch.BigEndian {
			p = add(p, 7)
		}
		stringStructOf(&str).str = p
		stringStructOf(&str).len = 1
		return
	}

	var p unsafe.Pointer
	if buf != nil && n <= len(buf) {
		p = unsafe.Pointer(buf)
	} else {
		p = mallocgc(uintptr(n), nil, false)
	}
	stringStructOf(&str).str = p
	stringStructOf(&str).len = n
	memmove(p, unsafe.Pointer(ptr), uintptr(n))
	return
}

```
## 测试

```go
package main

import (
	"fmt"
	"testing"
)

var s = "ABCD"

func TestBytesToString(t *testing.T) {
	fmt.Println(StringToBytes(s))
}

func TestStringToBytes(t *testing.T) {
	fmt.Println(StringToBytes1(s))
}

func TestStringToBytes1(t *testing.T) {
	fmt.Println(BytesToString([]byte(s)))
}

var c = "benchmark string to bytes"
var x = "hello world **********************************hello world ******************" +
	"****************hello world **********************************hello world **********************************hello world " +
	"**********************************hello world **********************************" +
	"****************hello world **********************************hello world **********************************hello world " +
	"****************hello world **********************************hello world **********************************hello world "

func BenchmarkStringToByteSmall(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		_ = StringToBytes(c)
	}
}

func BenchmarkStringToByte1Small(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		_ = StringToBytes1(c)
	}
}

func BenchmarkStringToBytesNormalSmall(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		_ = []byte(c)
	}
}

func BenchmarkStringToByte(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		_ = StringToBytes(x)
	}
}

func BenchmarkStringToByte1(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		_ = StringToBytes1(x)
	}
}

func BenchmarkStringToBytesNormal(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		_ = []byte(x)
	}
}

var d = []byte(c)

func BenchmarkBytesToStringSmall(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		_ = BytesToString(y)
	}
}

func BenchmarkBytesToStringNormalSmall(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		_ = string(y)
	}
}

var y = []byte(x)

func BenchmarkBytesToString(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		_ = BytesToString(y)
	}
}

func BenchmarkBytesToStringNormal(b *testing.B) {
	for i := 0; i <= b.N; i++ {
		_ = string(y)
	}
}
```
![{db0c5bfc-a0a9-4ff8-8eb9-4e1417929802}.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666578599555-f7d99a06-79a9-4e13-b875-be311f93343e.png#clientId=ud2544ff2-8ae6-4&from=paste&height=427&id=u3cb80b1b&name=%7Bdb0c5bfc-a0a9-4ff8-8eb9-4e1417929802%7D.png&originHeight=427&originWidth=981&originalType=binary&ratio=1&rotation=0&showTitle=false&size=59857&status=done&style=none&taskId=u6d875ab1-6e57-4b28-a6b3-1e0fd0ba2ec&title=&width=981)
