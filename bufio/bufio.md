
```go
// Package bufio implements buffered I/O. It wraps an io.Reader or io.Writer
// object, creating another object (Reader or Writer) that also implements
// the interface but provides buffering and some help for textual I/O.
package bufio

import (
	"bytes"
	"errors"
	"io"
	"strings"
	"unicode/utf8"
)

const (
	defaultBufSize = 4096
)

var (
	ErrInvalidUnreadByte = errors.New("bufio: invalid use of UnreadByte")
	ErrInvalidUnreadRune = errors.New("bufio: invalid use of UnreadRune")
	ErrBufferFull        = errors.New("bufio: buffer full")
	ErrNegativeCount     = errors.New("bufio: negative count")
)
```

`bufio` 包实现了有缓冲的 `I/O`. 它包装一个 `io.Reader` 或 `io.Writer` 接口对象, 创建了另一个对象（`Reader` 或 `Writer`）, 该对象也实现了接口, 且同时还提供了缓冲和一些文本 `I/O` 的帮助函数的对象.

### Reader对象

`bufio.Reader` 是 `bufio` 中对`io.Reader` 的封装.

```go
// Reader implements buffering for an io.Reader object.
type Reader struct {
	buf          []byte 
	rd           io.Reader // 底层的io.Reader
	r, w         int       // r:从buf中读走的字节（偏移）; w:buf中填充内容的偏移; 
	                       // w - r 是buf中可被读的长度（缓存数据的大小）, 也是Buffered()方法的返回值
	err          error
	lastByte     int // 最后一次读到的字节（ReadByte/UnreadByte)
	lastRuneSize int // 最后一次读到的Rune的大小(ReadRune/UnreadRune)
}
```

### Reader对象实例化

`bufio` 包提供了两个实例化 `bufio.Reader` 对象的函数, `NewReader` 和 `NewReaderSize`. 其中, `NewReader` 函数是调用 `NewReaderSize` 完成的.   
不同的是, `NewReaderSize` 可以通过 `size` 字段指定缓冲区的大小.

```go
const minReadBufferSize = 16
const maxConsecutiveEmptyReads = 100
```

`NewReaderSize` 如果参数 `io.Reader` 已经是一个足够大的 `Reader`, 它返回底层的 `Reader`

```go
// NewReaderSize returns a new Reader whose buffer has at least the specified
// size. If the argument io.Reader is already a Reader with large enough
// size, it returns the underlying Reader.
func NewReaderSize(rd io.Reader, size int) *Reader {
	// Is it already a Reader?
	b, ok := rd.(*Reader)
	if ok && len(b.buf) >= size {
		return b
	}
	if size < minReadBufferSize {
		size = minReadBufferSize
	}
	r := new(Reader)
	r.reset(make([]byte, size), rd)
	return r
}
```

```go
// NewReader returns a new Reader whose buffer has the default size.
func NewReader(rd io.Reader) *Reader {
    // defaultBufSize = 4096,默认的大小
	return NewReaderSize(rd, defaultBufSize)
}
```

`Size` 方法返回底层缓冲区的大小（以字节为单位）.

```go
// Size returns the size of the underlying buffer in bytes.
func (b *Reader) Size() int { return len(b.buf) }
```

`Reset` 方法丢弃所有缓冲数据, 重置所有状态, 并将缓冲读取器切换为从 `r` 读取.

```go
// Reset discards any buffered data, resets all state, and switches
// the buffered reader to read from r.
func (b *Reader) Reset(r io.Reader) {
	b.reset(b.buf, r)
}

func (b *Reader) reset(buf []byte, r io.Reader) {
	*b = Reader{
		buf:          buf,
		rd:           r,
		lastByte:     -1,
		lastRuneSize: -1,
	}
}
```

`fill` 方法从文件读取内容充满缓存区. 当缓冲区已经有数据时, 首先将现有未被读取的数据挪到开头, 并重新计算 `buf` 中填充内容的偏移.  
`fill` 方法将尝试读取 `maxConsecutiveEmptyReads` 次, 当对 `Read` 的多次调用都未能返回任何数据或错误时, 返回 `ErrNoProgress`.

```go
// fill reads a new chunk into the buffer.
func (b *Reader) fill() {
	// Slide existing data to beginning.
	if b.r > 0 {
		copy(b.buf, b.buf[b.r:b.w])
		b.w -= b.r
		b.r = 0
	}

	if b.w >= len(b.buf) {
		panic("bufio: tried to fill full buffer")
	}

	// Read new data: try a limited number of times.
	for i := maxConsecutiveEmptyReads; i > 0; i-- {
		n, err := b.rd.Read(b.buf[b.w:])
		if n < 0 {
			panic(errNegativeRead)
		}
		b.w += n
		if err != nil {
			b.err = err
			return
		}
		if n > 0 {
			return
		}
	}
	b.err = io.ErrNoProgress
}
```

`bufio.Read(p []byte)` 的思路如下.

* 当缓存区有内容的时, 将缓存区内容全部填入 `p` 并清空缓存区.
* 当缓存区没有内容的时候且 `len(p)>len(buf)`, 即要读取的内容比缓存区还要大, 直接去文件读取即可.
* 当缓存区没有内容的时候且 `len(p)<len(buf)`, 即要读取的内容比缓存区小, 缓存区从文件读取内容充满缓存区, 并将 `p` 填满（此时缓存区有剩余内容）.
* 以后再次读取时缓存区有内容, 将缓存区内容全部填入 `p` 并清空缓存区（此时和情况 1 一样）.

```go
// Read reads data into p.
// It returns the number of bytes read into p.
// The bytes are taken from at most one Read on the underlying Reader,
// hence n may be less than len(p).
// To read exactly len(p) bytes, use io.ReadFull(b, p).
// At EOF, the count will be zero and err will be io.EOF.
func (b *Reader) Read(p []byte) (n int, err error) {
	n = len(p)
	if n == 0 {
		if b.Buffered() > 0 {
			return 0, nil
		}
		return 0, b.readErr()
	}
	// r:从buf中读走的字节（偏移）；w:buf中填充内容的偏移；
	// w - r 是buf中可被读的长度（缓存数据的大小），也是Buffered()方法的返回值
	// b.r == b.w 表示，当前缓冲区里面没有内容
	if b.r == b.w {
		if b.err != nil {
			return 0, b.readErr()
		}
		// 如果p的大小大于等于缓冲区大小，则直接将数据读入p，然后返回
		if len(p) >= len(b.buf) {
			// Large read, empty buffer.
			// Read directly into p to avoid copy.
			n, b.err = b.rd.Read(p)
			if n < 0 {
				panic(errNegativeRead)
			}
			if n > 0 {
				b.lastByte = int(p[n-1])
				b.lastRuneSize = -1
			}
			return n, b.readErr()
		}
		// buff容量大于p，直接将buff中填满
		// One read.
		// Do not use b.fill, which will loop.
		b.r = 0
		b.w = 0
		n, b.err = b.rd.Read(b.buf)
		if n < 0 {
			panic(errNegativeRead)
		}
		if n == 0 {
			return 0, b.readErr()
		}
		b.w += n
	}

	// copy缓存区的内容到p中（填充满p）
	// copy as much as we can
	n = copy(p, b.buf[b.r:b.w])
	b.r += n
	b.lastByte = int(b.buf[b.r-1])
	b.lastRuneSize = -1
	return n, nil
}
```
