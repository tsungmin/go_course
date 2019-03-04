# 10 Defer and Error Handling

## Deferred function call

在 code block 或 function 結束前，一定要執行的程式碼。**defer** 的呼叫順序是 **stack** 的 LIFO (Last In First Out)，並且利用當下的變數值來執行。

```go {.line-numbers}
func double(x int) (result int) {
    defer func() { fmt.Printf("double(%d) = %d\n", x, result) }()
    return x + x
}

double(4) // double(4) = 8
```

在有關 I/O 處理時，一定會用到。

```go {.line-numbers}
func ReadFile(filename string) ([]byte, error) {
    f, err := os.Open(filename)

    if err != nil {
        return nil, err
    }

    defer f.Close()
    return ReadAll(f)
}
```

### defer in loop

在 loop 使用 defer 時，要注意：

1. defer 宣告的 function 會在離開 loop 時，才會執行，
1. 變數的 binding 時機點。

```go {.line-numbers}
package main

import "fmt"

func a1() {
    fmt.Println("\na1 loop start")
    for i := 0; i < 3; i++ {
        defer fmt.Print(i, " ")
    }
    fmt.Println("a1 loop end")
}

func a2() {
    fmt.Println("\na2 loop start")
    for i := 0; i < 3; i++ {
        defer func() {
            fmt.Print(i, " ") // warn: [go-vet] loop variable i captured by func literal
        }()
    }
    fmt.Println("a2 loop end")
}

func a3() {
    fmt.Println("\na3 loop start")
    for i := 0; i < 3; i++ {
        defer func(n int) {
            fmt.Print(n, " ")
        }(i)
    }
    fmt.Println("a3 loop end")
}

func main() {
    for i := 0; i < 3; i++ {
        defer fmt.Printf("\n%d main end", i)
    }

    a1() // 2 1 0
    a2() // 3 3 3
    a3() // 2 1 0
}
```

1. 在 main 的 loop 宣告的 defer 函式，會在 main 結束前執行。
1. 在 a1, a2, a3 的 loop 宣告的 defer function，會在各別 function 結束前執行。
1. a1: 使用當下迴圈 i 的變數值。因此會是 **2 1 0**
1. a2: 每次迴圈完成時，會記錄要執行一個 **anonymous function**，當迴圈結束後，則開始執行 defer 記錄的 function，此時 i 的值已經是 **3**。
1. 與 a2 類似，多傳入當下 i 的值，因此結果與會 a1 相同。

使用 **defer** 要特別小心被呼叫的時機點與綁定的變數值。

### defer 與 os.Exit

當使用 `os.Exit` 時，設定的 defer function 並**不會被執行**。因此在宣告 defer 之後，就不要再用 `os.Exit` 中斷程式。

```go {.line-numbers}
defer fmt.Println("call defer")
fmt.Println("main end")
os.Exit(0)
```

## Error Handling, Panic, Revcover

### errors

`error` 是 Go 內建的 data type，它是一個 interface, 定義如下：

```go {.line-numbers}
type error interface {
    Error() string
}
```

因此，要自定義 error，只要實作這個 interface 即可。

```go {.line-numbers}
type MyError struct {
    Code int
    Message string
}

func (e *MyError) Error() string {
    return fmt.Sprintf("%d: %s", e.Code, e.Message)
}
```

在 Go 的 function 設計中，很多都會回傳包含 error 的 tuple。eg:

```go {.line-numbers}
resp, err := http.Get(url)

if err != nil {
    return nil, err
}
```

在 Go 的 Code style 規範中，如果 function 會回傳 tuple 且含有 error 時，請把 error 放在 tuple 最後一個欄位。

```go {.line-numbers}
func FindMember(id int) (*Member, error)
```

### Panic

`panic` 會導致程式中斷。在 Go 的設計中，除非是很嚴重的錯誤，才會使用 **panic**，如像 I/O, 設定檔錯誤等。如是預期到，在撰寫程式時，則儘量檢查並用 **error** 來處理。

Panic 現象：

```go {.line-numbers}
func f(x int) {
    fmt.Printf("f(%d)\n", x+0/x) // panics if x == 0
    defer fmt.Printf("defer %d\n", x)
    f(x - 1)
}

f(3)
```

Output:

```text
f(3)
f(2)
f(1)
defer 1
defer 2
defer 3
panic: runtime error: integer divide by zero

goroutine 1 [running]:
main.f(0x0)
    /Users/kigi/Data/cyberon/go/src/go_course/class10/ex10-04/main.go:6 +0x16f
main.f(0x1)
    /Users/kigi/Data/cyberon/go/src/go_course/class10/ex10-04/main.go:8 +0x14a
main.f(0x2)
    /Users/kigi/Data/cyberon/go/src/go_course/class10/ex10-04/main.go:8 +0x14a
main.f(0x3)
    /Users/kigi/Data/cyberon/go/src/go_course/class10/ex10-04/main.go:8 +0x14a
main.main()
    /Users/kigi/Data/cyberon/go/src/go_course/class10/ex10-04/main.go:12 +0x2a
exit status 2
```

不應該使用 Panic 的案例，請回傳 **error**:

```go {.line-numbers}
func Reset(x *Buffer) {
    if x == nil {
        panic("x is nil") // unnecessary!
    }
    x.elements = nil
}
```

### Recover

用在取得 panic 發生的原因，通常與 **defer** 撘配使用，用在 debug 執行時期(runtime)的錯誤。

eg:

```go {.line-numbers}
package main

import "fmt"

func f(x int) {
    fmt.Printf("f(%d)\n", x+0/x) // panics if x == 0
    defer fmt.Printf("defer %d\n", x)
    f(x - 1)
}

func main() {
    defer func() {
        if p := recover(); p != nil {
            fmt.Printf("internal error: %v\n", p)
        }
    }()

    f(3)
}
```

output:

```text
f(3)
f(2)
f(1)
defer 1
defer 2
defer 3
internal error: runtime error: integer divide by zero
```