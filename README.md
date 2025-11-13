# Go: Performance Tuning and Benchmarking

## Benchmarking basics

### What is benchmarking

- Performance measure
- Make data-driven decisions

### Go test tool

- Deterministic performances
- Enviroment should be quiet
- Constant results

#### Command line options

- bench: which test
- cpu: parallel CPUs
- benchtime: elapsed time
- test: execution

```sh
go test -run ^$ -bench
go test -run none -bench
```

### Benchmarking in Go

- Average time
- Enough time
- Iterations

#### Benchtime

- Timers: StopTimer, ResetTimer
- Dfaulted to 1s

### Benchmark structure

```go
package fibonacci

import (
	"fmt"
	"testing"
)

type Sequence struct {
	n    int
}

func (s *Sequence) fib(n int) int {
	switch n {
	case 0:
		return 0
	case 1:
		return 1
	}
	return s.fib(n-1) + s.fib(n-2)
}

func (s *Sequence) Next() int {
	s.n++
	return s.fib(s.n)
}

func (s *Sequence) N(n int) int {
	return s.fib(n)
}

// Our benchmark shows that for fib(n), as n increases, so does the run time!
//
// This progression doesn't seem to be linear -- it seems exponential in fact.
//
// Can we modify the Sequence type to improve performance?  Would it help if
// the function memoized previous answers?

func BenchmarkFibonacci(b *testing.B) {
  var seq Sequence
	for n := 0; n < 5; n++ {
		b.Run(fmt.Sprintf("fib %d", n), func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				seq.N(n)
			}
		})
	}
}

func TestFibonacci(t *testing.T) {
	fibonacci := []int{0, 1, 1, 2, 3, 5, 8, 13, 21}

	var seq Sequence

	for i := 0; i < len(fibonacci); i++ {
		t.Run(fmt.Sprintf("fib %d", i), func(t *testing.T) {
			if got, expected := seq.N(i), fibonacci[i]; got != expected {
				t.Fatalf("expected fib(%d) to be %d; got %d", i, expected, got)
			}
		})
	}
}
```

```sh
go test -run none -bench BenchmarkN
```

## Managing benchmark timer

### Benchmark timer

- b.StopTimer
- b.startTimer
- b.ResetTimer

```sh
go test -bench .
```

```go
func BenchmarkCount(b *testing.B) {
	words, err := LoadFile("testdata/alice-in-wonderland.txt")
	if err != nil {
		b.Fatal(err)
	}
	for i := 0; i < b.N; i++ {
		Count(allWords)
	}
}
```

### Cleaning up

```go
package cleanup

import (
	"fmt"
	"math/rand"
	"os"
	"path"
	"testing"
)

func BenchmarkWriteBytes(b *testing.B) {
	b.StopTimer()

	dir, err := os.MkdirTemp("testdata", "write*")
	if err != nil {
		b.Fatal(err)
	}

	for i := 0; i < b.N; i++ {
		var buf [16]byte
		rand.Read(buf[:])
		fn := path.Join(dir, fmt.Sprintf("%x", buf[:]))

		out, err := os.Create(fn)
		if err != nil {
			b.Fatal(err)
		}
		defer out.Close()
		message := "benchmarking in go!"
		b.StartTimer()
		fmt.Fprintf(out, message)
		b.StopTimer()
		b.SetBytes(int64(len(message)))
	}
}
```

### Table driven

```go
package rand

import (
	cryptorand "crypto/rand"
	mathrand "math/rand"
	"testing"
)

func BenchmarkRandRead(b *testing.B) {
	funcs := map[string]func([]byte) (int, error){
		"mathrand":   mathrand.Read,
		"cryptorand": cryptorand.Read,
	}

	for name, f := range funcs {
		buf := make([]byte, 128)
		b.Run(name, func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				f(buf)
			}
		})
	}
}

func BenchmarkRandReadInterleved(b *testing.B) {
	funcs := map[string]func([]byte) (int, error){
		"mathrand":   mathrand.Read,
		"cryptorand": cryptorand.Read,
	}

	for name, f := range funcs {
		buf := make([]byte, 128)
		b.Run(name, func(b *testing.B) {
			b.StopTimer()
			for i := 0; i < b.N; i++ {
				b.StartTimer()
				f(buf)
				b.StopTimer()
			}
		})
	}
}
```

### Parallel benchmark

```sh
go test -run none -bench . -cpu 1,2,4,6
```

## Benchmark accurancy

### Inlined functions
