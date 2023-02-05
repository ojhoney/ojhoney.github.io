---
title: "[Go][Python] PS를 위한 입출력"
date: 2022-11-08T21:32:33+09:00
draft: false
---

## Intro

알고리즘 문제풀이 시, 우선적으로 하나의 테스트 케이스를 상정하고 문제를 풀어나가는 경우가 많다. debugger를 실행할 때마다 매번 같은 테스트케이스를 복붙하는 것은 매우 귀찮은 일이다. 또한, Notebook 환경에서 표준입력은 사용하기 매우 번거롭다. 

이 문제에 대한 해결방법으로 테스트 케이스를 문자열 변수 `sample`로 저장하고 처리하는 방법에 대해 소개한다.

## Python

빠른 입력을 위하여 `sys.stdin.readline` 함수를 사용하는 경우가 많다. 한 줄 씩 읽어오므로 이에 맞게 테스트 케이스 문자열 또한 수정해준다. 

```python
# 따옴표 세 개로 묶으면 개행이 포함된 문자열을 표현할 수 있다
sample = """5
1 2 3 4 5""" 
sample = iter(sample.split('\n'))
sample = iter(sample.splitline(True))
def read(): return next(sample)

import sys
# read = sys.stdin.readline

if __name__ == "__main__":
	N = int(read()) 
	# N = 5
	arr = list(map(int, read().split())) 
	# arr = [1, 2, 3, 4, 5]
```

문자열 변수 `sample` 를 반복자(iterator)로 만들어서 읽을 때마다 한 줄 씩 읽어준다.

답안 제출 시 1~4줄을 지우고 n줄의 주석처리를 취소하여 `read`를 `sys.stdin.readline` 의 alias로 사용하면 된다.

## Go

빠른 입력을 위하여 `bufio.Scanner`를 사용한다.  

```go
func main() {
	defer wr.Flush()

	N := scanInt()
	// N = 5
	arr := make([]int, N)
	for i := 0; i < N; i++ {
		arr[i] = scanInt()
	}
	// arr = []int{1, 2, 3, 4, 5}

}

//---------
// Fast IO
//---------

var (
	sc *bufio.Scanner
	wr *bufio.Writer
)

func init() {
	// back quote(`)로 묶으면 개행이 포함된 문자열을 표현할 수 있다.
	sample := `5
	1 2 3 4 5`

	sc = bufio.NewScanner(strings.NewReader(sample))
	//sc = bufio.NewScanner(os.Stdin)
	wr = bufio.NewWriter(os.Stdout)

	sc.Split(bufio.ScanWords)
}

func scanInt() int {
	sc.Scan()
	ret, _ := strconv.Atoi(sc.Text())
	return ret
}
```

`init` 함수에서 입출력과 테스트 케이스 문자열 변수를 지정하는게 깔끔하다. 

`bufio.NewScanner`는 `io.Reader`를 인자로 받는데 이를 위해 `strings.NewReader` 함수로 문자열 변수 `sample` 을 알맞게 변형해준다.  

답안 제출 시 `init` 함수 안의 n~m 줄을 지우고 `sc` 의 주석처리를 지우면 된다. 또한, `import` 문에 `strings` 패키지를 지워줘야 한다.