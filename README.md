Buffer 
==========

[![GoDoc](https://godoc.org/github.com/djherbis/buffer?status.svg)](https://godoc.org/github.com/djherbis/buffer)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg)](LICENSE.txt)
[![Build Status](https://travis-ci.org/djherbis/buffer.svg?branch=master)](https://travis-ci.org/djherbis/buffer) 
[![Coverage Status](https://coveralls.io/repos/djherbis/buffer/badge.svg?branch=master)](https://coveralls.io/r/djherbis/buffer?branch=master)
[![Join the chat at https://gitter.im/djherbis/buffer](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/djherbis/buffer?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Usage
------------

The following buffers provide simple unique behaviours which when composed can create complex buffering strategies. For use with github.com/djherbis/nio for Buffered io.Pipe and io.Copy implementations.

For example: 

```go
import (
  "github.com/djherbis/buffer"
  "github.com/djherbis/nio"
  
  "io/ioutil"
)

// Buffer 32KB to Memory, after that buffer to 100MB chunked files
buf := buffer.NewUnboundedBuffer(32*1024, 100*1024*1024)
nio.Copy(w, r, buf) // Reads from r, writes to buf, reads from buf writes to w (concurrently).

// Buffer 32KB to Memory, discard overflow
buf = buffer.NewSpill(32*1024, ioutil.Discard)
nio.Copy(w, r, buf)
```

Supported Buffers
------------

#### Bounded Buffers ####

Memory: Wrapper for bytes.Buffer

File: File-based buffering. The file never exceeds Cap() in length, no matter how many times its written/read from. It accomplishes this by "wrapping" around the fixed max-length file when the data gets too long but there is available freed space at the beginning of the file. The caller is responsible for closing and deleting the file when done.

```go
import (
  "ioutil"
  "os"
  
  "github.com/djherbis/buffer"
)

// Create a File-based Buffer with max size 100MB
file, err := ioutil.TempFile("", "buffer")
if err != nil {
	return err
}
defer os.Remove(file.Name())
defer file.Close()

buf := buffer.NewFile(100*1024*1024, file)

// A simpler way:
pool := NewFilePool(100*1024*1024, "") // "" -- use temp dir
buf := pool.Get()   // allocate the buffer
defer pool.Put(buf) // close and remove the allocated file for the buffer

```

Multi: A fixed length linked-list of buffers. Each buffer reads from the next buffer so that all the buffered data is shifted upwards in the list when reading. Writes are always written to the first buffer in the list whose Len() < Cap().

```go
import (
  "github.com/djherbis/buffer"
)

mem  := buffer.New(32*1024)
file := buffer.NewFile(100*1024*1024, someFileObj)) // you'll need to manage Open(), Close() and Delete someFileObj

// Buffer composed of 32KB of memory, and 100MB of file.
buf := buffer.NewMulti(mem, file)
```

#### Unbounded Buffers ####

Partition: A queue of buffers. Writes always go to the first buffer in the queue which isn't full. If all buffers are full, a new buffer is "pushed" to the end of the queue (generated by a user-given function). Reads come from the first buffer, when the first buffer is emptied it is "popped" off the queue.

```go
import (
  "github.com/djherbis/buffer"
)

// Create 32 KB sized-chunks of memory as needed to expand/contract the buffer size.
buf := buffer.NewPartition(NewMemPool(32*1024))

// Create 100 MB sized-chunks of files as needed to expand/contract the buffer size.
buf = buffer.NewPartition(NewFilePool(100*1024*1024, ""))
```

Ring: A single buffer which begins overwriting the oldest buffered data when it reaches its capacity.

```go
import (
  "github.com/djherbis/buffer"
)

// Create a File-based Buffer with max size 100MB
file := buffer.NewFile(100*1024*1024, someFileObj) // you'll need to Open(), Close() and Delete someFileObj.

// If buffered data exceeds 100MB, overwrite oldest data as new data comes in
buf := buffer.NewRing(file) // requires BufferAt interface.
```

Spill: A single buffer which when full, writes the overflow to a given io.Writer.
-> Note that it will actually "spill" whenever there is an error while writing, this should only be a "full" error.

```go
import (
  "github.com/djherbis/buffer"
  "github.com/djherbis/nio"
  
  "io/ioutil"
)

// Buffer 32KB to Memory, discard overflow
buf := buffer.NewSpill(32*1024, ioutil.Discard)
nio.Copy(w, r, buf)
```

#### Empty Buffer ####

Discard: Reads always return EOF, writes goto ioutil.Discard.

```go
import (
  "github.com/djherbis/buffer"
)

// Reads will return io.EOF, writes will return success (nil error, full write) but no data was written.
buf := buf.NewDiscard()
```

Custom Buffers
------------

Feel free to implement your own buffer, just meet the required interface (Buffer/BufferAt) and compose away!

```go

// Buffer Interface used by Multi and Partition
type Buffer interface {
	Len() int64
	Cap() int64
	io.Reader
	io.Writer
	Reset()
}

// BufferAt interface used by Ring
type BufferAt interface {
	Buffer
	io.ReaderAt
	io.WriterAt
}

```

Installation
------------
```sh
go get github.com/djherbis/buffer
```
