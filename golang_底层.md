# 变量逃逸的概念

go语言一般函数内部定义的变量会在栈上分配内存，但如果变量通过指针传递给了函数外部的介质，则称为发生了逃逸现象，那么此变量最终会在堆中分配内存。

```go
func toHeap() *int {
	var x int
	return &x
}

func toStack() int {
	x := new(int)
	*x = 1
	return *x
}

func main() {

}
```

```bash
go run -gcflags '-m -l' main.go
# command-line-arguments
./main.go:4:6: moved to heap: x
./main.go:9:10: toStack new(int) does not escape
```

和c语言不同，c语言没有自带逃逸功能

```c
#include <stdio.h>

char *b = NULL;

void foo()
{
    int a = 5;
    char c[15] = "sdfadfaf\n";
    b = c;
    return;
}

int main ()
{
    foo();
    printf("value: %s\n", b);
    return 0;
}
```

```bash
value:  R_??N?
```


