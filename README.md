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

```go
package inline

import (
	"math/rand"
	"testing"
	"runtime"
)

func Add(x, y int) int {
	return x + y
}

func BenchmarkAdd(b *testing.B) {
	x, y := rand.Int(), rand.Int()
	var sum int
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		sum += Add(x, y)
	}
	runtime.KeepAlive(&sum)
}
```

### Forgetting iterate

```go
package noiterate

import (
	"math"
	"math/rand"
	"testing"
)

//go:noinline
func Hypot(a, b float64) float64 {
	return math.Sqrt(a*a + b*b)
}

func BenchmarkWithoutIterating(b *testing.B) {
	x, y := rand.Float64(), rand.Float64()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		Hypot(x,y)
	}
}
```

### Bad wall

```go
package ratio

import (
	"fmt"
	"io/fs"
	"os"
	"testing"
)

//go:noinline
func Add(x, y int) int {
	return x + y
}

func BenchmarkBadRatio(b *testing.B) {
	// walk a file system, open a text file, read the contents, then benchmark adding them.

	for i := 0; i < b.N; i++ {
		b.StopTimer()
		root := os.DirFS("testdata/numbers")
		fs.WalkDir(root, ".", func(p string, d fs.DirEntry, err error) error {
			if d.IsDir() {
				return nil
			}

			in, err := root.Open(p)
			if err != nil {
				b.Fatal(err)
			}
			defer in.Close()

			var x, y int
			if n, err := fmt.Fscanf(in, "%d, %d\n", &x, &y); n != 2 || err != nil {
				b.Fatalf("could not read integer pair from %q", p)
			}

			b.StartTimer()
			Add(x, y)
			b.StopTimer()
			return nil
		})
	}
}
```

### Challenge: optimizing ID

```go
package crockford

import (
	"testing"
)

func TestNewID(t *testing.T) {
	t.Logf("NewID() = %s", NewID())
}

func TestEncode(t *testing.T) {
	tests := map[string]struct {
		input    []byte
		expected string
	}{
		"empty string": {},
		"simple": {
			input:    []byte("The quick brown fox jumps over the lazy dog."),
			expected: "AHM6A83HENMP6TS0C9S6YXVE41K6YY10D9TPTW3K41QQCSBJ41T6GS90DHGQMY90CHQPEBG=",
		},
	}

	for name, test := range tests {
		test := test
		t.Run(name, func(t *testing.T) {
			t.Parallel()
			if got, expected := Encode(test.input), test.expected; got != expected {
				t.Fatalf("Expected Encode(%q) to return %q; got %q", test.input, expected, got)
			}
		})
	}
}
```

## Comparing benchmarks

### Using benchstat

```go
package fib

import (
	"strconv"
	"testing"
)

func BenchmarkFib(b *testing.B) {
	seq := New()
	for n := 0; n < 17; n++ {
		b.Run(strconv.Itoa(n), func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				seq.fib(n)
			}
		})
	}
}

func TestFib(t *testing.T) {
	tests := []int{0, 1, 1, 2, 3, 5, 8, 13, 21}
	seq := New()
	for i, expected := range tests {
		i, value := i, expected
		t.Run(strconv.Itoa(i), func(t *testing.T) {
			if got, expected := seq.fib(i), value; got != expected {
				t.Fatalf("fib(%v) returned %v; expected %v", i, got, expected)
			}
		})
	}
}
```
