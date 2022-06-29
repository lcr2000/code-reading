
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

## Reader 对象

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

### Reader 对象实例化

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

### Size 方法

`Size` 方法返回底层缓冲区的大小（以字节为单位）.

```go
// Size returns the size of the underlying buffer in bytes.
func (b *Reader) Size() int { return len(b.buf) }
```

### Reset 方法

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

### 内部 fill 方法

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

### Peek 方法

`Peek` 方法在不推进 `r` 的情况下, 返回 `Reader` 中 `r` 后面可读取的 `n` 个字节, 也就是前 `n` 个 未读字节

```go
// Peek returns the next n bytes without advancing the reader. The bytes stop
// being valid at the next read call. If Peek returns fewer than n bytes, it
// also returns an error explaining why the read is short. The error is
// ErrBufferFull if n is larger than b's buffer size.
//
// Calling Peek prevents a UnreadByte or UnreadRune call from succeeding
// until the next read operation.
func (b *Reader) Peek(n int) ([]byte, error) {
	if n < 0 {
		return nil, ErrNegativeCount
	}

	b.lastByte = -1
	b.lastRuneSize = -1

	for b.w-b.r < n && b.w-b.r < len(b.buf) && b.err == nil {
		b.fill() // b.w-b.r < len(b.buf) => buffer is not full
	}

	// 边界条件放这里校验是因为: 即使 n 大于缓冲区的情况下, 也要把 数据读出来返回给调用者, 并且返回具体错误的原因
	if n > len(b.buf) {
		return b.buf[b.r:b.w], ErrBufferFull
	}

	// 0 <= n <= len(b.buf)
	var err error
	if avail := b.w - b.r; avail < n {
		// not enough data in buffer
		n = avail
		err = b.readErr()
		if err == nil {
			err = ErrBufferFull
		}
	}
	return b.buf[b.r : b.r+n], err
}
```

### Discard 方法

`Discard` 方法把已读取游标进行 `+n`, 也就是跳过接下来的 n 个字节数.    
如果 `Discard` 可以跳过的字节数少于 `n` 个, 它会返回一个错误；如果 `0 <= n <= b.Buffered()`, 则无需从底层 `io.Reader` 读取.

```go
// Discard skips the next n bytes, returning the number of bytes discarded.
//
// If Discard skips fewer than n bytes, it also returns an error.
// If 0 <= n <= b.Buffered(), Discard is guaranteed to succeed without
// reading from the underlying io.Reader.
func (b *Reader) Discard(n int) (discarded int, err error) {
	if n < 0 {
		return 0, ErrNegativeCount
	}
	if n == 0 {
		return
	}
	remain := n
	for {
		skip := b.Buffered()
		if skip == 0 {
			b.fill()
			skip = b.Buffered()
		}
		if skip > remain {
			skip = remain
		}
		b.r += skip
		remain -= skip
		if remain == 0 {
			return n, nil
		}
		if b.err != nil {
			return n - remain, b.readErr()
		}
	}
}
```

### Read 方法

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

### ReadByte 方法

`ReadByte` 读取并返回一个字节, 如果没有可用的数据, 会返回错误. 同时会将 `r` 游标递增, 将 `lastByte` 设置为读取到的单个字节.

```go
// ReadByte reads and returns a single byte.
// If no byte is available, returns an error.
func (b *Reader) ReadByte() (byte, error) {
	b.lastRuneSize = -1
	for b.r == b.w {
		if b.err != nil {
			return 0, b.readErr()
		}
		b.fill() // buffer is empty
	}
	c := b.buf[b.r]
	b.r++
	b.lastByte = int(c)
	return c, nil
}
```

### UnreadByte 方法

`UnreadByte` 还原最近一次读取操作读出的最后一个字节, 相当于让读偏移量 `r` 前移一个字节. 连续两次 `UnreadByte` 操作, 而中间没有任何读取操作会返回错误

```go
// UnreadByte unreads the last byte. Only the most recently read byte can be unread.
//
// UnreadByte returns an error if the most recent method called on the
// Reader was not a read operation. Notably, Peek is not considered a
// read operation.
func (b *Reader) UnreadByte() error {
	if b.lastByte < 0 || b.r == 0 && b.w > 0 {
		return ErrInvalidUnreadByte
	}
	// b.r > 0 || b.w == 0
	if b.r > 0 {
		b.r--
	} else {
		// b.r == 0 && b.w == 0
		b.w = 1
	}
	b.buf[b.r] = byte(b.lastByte)
	b.lastByte = -1
	b.lastRuneSize = -1
	return nil
}
```

### ReadRune 方法

`ReadRune` 读取一个 `utf-8` 编码的 `unicode` 码值, 返回该码值、其编码长度和可能的错误.   
如果 `utf-8` 编码非法, 读取位置只移动 `1` 字节, 返回 `U+FFFD`, 返回值 `size` 为 `1` 而 `err` 为 `nil`. 如果没有可用的数据, 会返回错误

```go
// ReadRune reads a single UTF-8 encoded Unicode character and returns the
// rune and its size in bytes. If the encoded rune is invalid, it consumes one byte
// and returns unicode.ReplacementChar (U+FFFD) with a size of 1.
func (b *Reader) ReadRune() (r rune, size int, err error) {
	for b.r+utf8.UTFMax > b.w && !utf8.FullRune(b.buf[b.r:b.w]) && b.err == nil && b.w-b.r < len(b.buf) {
		b.fill() // b.w-b.r < len(buf) => buffer is not full
	}
	b.lastRuneSize = -1
	if b.r == b.w {
		return 0, 0, b.readErr()
	}
	r, size = rune(b.buf[b.r]), 1
	if r >= utf8.RuneSelf {
		r, size = utf8.DecodeRune(b.buf[b.r:b.w])
	}
	b.r += size
	b.lastByte = int(b.buf[b.r-1])
	b.lastRuneSize = size
	return r, size, nil
}
```

### UnreadRune 方法

`UnreadRune` 还原前一次 `ReadRune` 操作读取的 `unicode` 码值, 相当于让读偏移量 `r` 前移一个码值长度.   
需要注意的是,  `UnreadRune` 方法调用前必须调用 `ReadRune` 方法, `UnreadRune` 比 `UnreadByte` 严格很多.

```go
// UnreadRune unreads the last rune. If the most recent method called on
// the Reader was not a ReadRune, UnreadRune returns an error. (In this
// regard it is stricter than UnreadByte, which will unread the last byte
// from any read operation.)
func (b *Reader) UnreadRune() error {
	if b.lastRuneSize < 0 || b.r < b.lastRuneSize {
		return ErrInvalidUnreadRune
	}
	b.r -= b.lastRuneSize
	b.lastByte = -1
	b.lastRuneSize = -1
	return nil
}
```

### Buffered 方法

`Buffered` 返回当前缓冲区未读取的字节数.

```go
// Buffered returns the number of bytes that can be read from the current buffer.
func (b *Reader) Buffered() int { return b.w - b.r }
```

### ReadSlice 方法

`ReadSlice` 查找 `delim` 并返回 `delim` 及其之前的所有数据的切片, 该操作会读出数据, 返回的切片是已读出数据的引用.   
如果 `ReadSlice` 在找到 `delim` 之前遇到错误, 则读出缓存中的所有数据并返回, 同时返回遇到 `error`（通常是 `io.EOF`）   
如果在整个缓存中都找不到 `delim`, 则返回 `ErrBufferFull`   
如果 `ReadSlice` 能找到 `delim`, 则返回 `nil`   
注意: 因为返回的 `Slice` 数据有可能被下一次读写操作修改, 因此大多数操作应该使用 `ReadBytes` 或 `ReadString`, 它们返回数据 `copy`   
不推荐!

```go
// ReadSlice reads until the first occurrence of delim in the input,
// returning a slice pointing at the bytes in the buffer.
// The bytes stop being valid at the next read.
// If ReadSlice encounters an error before finding a delimiter,
// it returns all the data in the buffer and the error itself (often io.EOF).
// ReadSlice fails with error ErrBufferFull if the buffer fills without a delim.
// Because the data returned from ReadSlice will be overwritten
// by the next I/O operation, most clients should use
// ReadBytes or ReadString instead.
// ReadSlice returns err != nil if and only if line does not end in delim.
func (b *Reader) ReadSlice(delim byte) (line []byte, err error) {
	s := 0 // search start index
	for {
		// Search buffer.
		if i := bytes.IndexByte(b.buf[b.r+s:b.w], delim); i >= 0 {
			i += s
			line = b.buf[b.r : b.r+i+1]
			b.r += i + 1
			break
		}

		// Pending error?
		if b.err != nil {
			line = b.buf[b.r:b.w]
			b.r = b.w
			err = b.readErr()
			break
		}

		// Buffer full?
		if b.Buffered() >= len(b.buf) {
			b.r = b.w
			line = b.buf
			err = ErrBufferFull
			break
		}

		s = b.w - b.r // do not rescan area we scanned before

		b.fill() // buffer is not full
	}

	// Handle last byte, if any.
	if i := len(line) - 1; i >= 0 {
		b.lastByte = int(line[i])
		b.lastRuneSize = -1
	}

	return
}
```

### ReadLine 方法

`ReadLine` 是一个低级的原始的行读取操作, 一般应该使用 `ReadBytes('\n')` 或 `ReadString('\n')`.   
ReadLine 通过调用 `ReadSlice` 方法实现, 返回的也是"引用", 回一行数据, 不包括行尾标记（`\n` 或 `\r\n`）.   
如果 在缓存中找不到行尾标记, 设置 `isPrefix` 为 `true`, 表示查找未完成.   
如果 在当前缓存中找到行尾标记, 将 `isPrefix` 设置为 `false`, 表示查找完成.   
如果 `ReadLine` 无法获取任何数据, 则返回一个错误信息（通常是 `io.EOF`）.   
不推荐!   

```go
// ReadLine is a low-level line-reading primitive. Most callers should use
// ReadBytes('\n') or ReadString('\n') instead or use a Scanner.
//
// ReadLine tries to return a single line, not including the end-of-line bytes.
// If the line was too long for the buffer then isPrefix is set and the
// beginning of the line is returned. The rest of the line will be returned
// from future calls. isPrefix will be false when returning the last fragment
// of the line. The returned buffer is only valid until the next call to
// ReadLine. ReadLine either returns a non-nil line or it returns an error,
// never both.
//
// The text returned from ReadLine does not include the line end ("\r\n" or "\n").
// No indication or error is given if the input ends without a final line end.
// Calling UnreadByte after ReadLine will always unread the last byte read
// (possibly a character belonging to the line end) even if that byte is not
// part of the line returned by ReadLine.
func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error) {
	line, err = b.ReadSlice('\n')
	if err == ErrBufferFull {
		// Handle the case where "\r\n" straddles the buffer.
		if len(line) > 0 && line[len(line)-1] == '\r' {
			// Put the '\r' back on buf and drop it from line.
			// Let the next call to ReadLine check for "\r\n".
			if b.r == 0 {
				// should be unreachable
				panic("bufio: tried to rewind past start of buffer")
			}
			b.r--
			line = line[:len(line)-1]
		}
		return line, true, nil
	}

	if len(line) == 0 {
		if err != nil {
			line = nil
		}
		return
	}
	err = nil

	if line[len(line)-1] == '\n' {
		drop := 1
		if len(line) > 1 && line[len(line)-2] == '\r' {
			drop = 2
		}
		line = line[:len(line)-drop]
	}
	return
}
```

### ReadBytes 方法

`ReadBytes` 查找 `delim` 并读出 `delim` 及其之前的所有数据.   
如果 `ReadBytes` 在找到 `delim` 之前遇到错误, 则返回遇到错误之前的所有数据, 同时返回遇到的错误（通常是 `io.EOF`）.   
如果 `ReadBytes` 找不到 `delim` 时, `err != nil`.   
返回的是数据的 `copy`, 不是引用.   

```go
// ReadBytes reads until the first occurrence of delim in the input,
// returning a slice containing the data up to and including the delimiter.
// If ReadBytes encounters an error before finding a delimiter,
// it returns the data read before the error and the error itself (often io.EOF).
// ReadBytes returns err != nil if and only if the returned data does not end in
// delim.
// For simple uses, a Scanner may be more convenient.
func (b *Reader) ReadBytes(delim byte) ([]byte, error) {
	full, frag, n, err := b.collectFragments(delim)
	// Allocate new buffer to hold the full pieces and the fragment.
	buf := make([]byte, n)
	n = 0
	// Copy full pieces and fragment in.
	for i := range full {
		n += copy(buf[n:], full[i])
	}
	copy(buf[n:], frag)
	return buf, err
}
```

### ReadString 方法

`ReadString` 返回的是字符串, 不是bytes.

```go
// ReadString reads until the first occurrence of delim in the input,
// returning a string containing the data up to and including the delimiter.
// If ReadString encounters an error before finding a delimiter,
// it returns the data read before the error and the error itself (often io.EOF).
// ReadString returns err != nil if and only if the returned data does not end in
// delim.
// For simple uses, a Scanner may be more convenient.
func (b *Reader) ReadString(delim byte) (string, error) {
	full, frag, n, err := b.collectFragments(delim)
	// Allocate new buffer to hold the full pieces and the fragment.
	var buf strings.Builder
	buf.Grow(n)
	// Copy full pieces and fragment in.
	for _, fb := range full {
		buf.Write(fb)
	}
	buf.Write(frag)
	return buf.String(), err
}
```

## Writer 对象

```go
// buffered output

// Writer implements buffering for an io.Writer object.
// If an error occurs writing to a Writer, no more data will be
// accepted and all subsequent writes, and Flush, will return the error.
// After all data has been written, the client should call the
// Flush method to guarantee all data has been forwarded to
// the underlying io.Writer.
type Writer struct {
	err error
	buf []byte
	n   int
	wr  io.Writer
}
```

和 `Reader` 类型一样, `bufio` 包提供了两个实例化 `bufio.Writer` 对象的函数：`NewWriter` 和 `NewWriterSize`.    
其中, `NewWriter` 函数是调用 `NewWriterSize` 函数实现的.

```go
// NewWriterSize returns a new Writer whose buffer has at least the specified
// size. If the argument io.Writer is already a Writer with large enough
// size, it returns the underlying Writer.
func NewWriterSize(w io.Writer, size int) *Writer {
	// Is it already a Writer?
	b, ok := w.(*Writer)
	if ok && len(b.buf) >= size {
		return b
	}
	if size <= 0 {
		size = defaultBufSize
	}
	return &Writer{
		buf: make([]byte, size),
		wr:  w,
	}
}

// NewWriter returns a new Writer whose buffer has the default size.
func NewWriter(w io.Writer) *Writer {
	return NewWriterSize(w, defaultBufSize)
}
```

### Size 方法

`Size` 返回底层缓冲区的大小（以字节为单位）.

```go
// Size returns the size of the underlying buffer in bytes.
func (b *Writer) Size() int { return len(b.buf) }
```

`Reset` 方法丢弃所有缓冲数据, 重置所有状态, 并将缓冲读取器切换为从 `r` 读取.

```go
// Reset discards any unflushed buffered data, clears any error, and
// resets b to write its output to w.
func (b *Writer) Reset(w io.Writer) {
	b.err = nil
	b.n = 0
	b.wr = w
```