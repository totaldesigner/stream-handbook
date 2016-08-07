# stream-handbook

이 문서는 [streams](http://nodejs.org/docs/latest/api/stream.html)  [node.js](http://nodejs.org/) 로 작성하기 위한 기본적인 내용을 다룬다. 

# node packaged manuscript

아래와 같이 npm 을 통해 설치할 수 있다.
```
npm install -g stream-handbook
```

`$PAGER` 에 있는 readme 파일을 열면 `stream-handbook` 명령어를 사용할 수 있다.

# introduction

```
"정원의 호스를 연결하는 것처럼 프로그램, 데이터, IO 를 연결할 수 있어야 한다."
```

[Doug McIlroy. October 11, 1964](http://cm.bell-labs.com/who/dmr/mdmpipe.html)

![doug mcilroy](http://substack.net/images/mcilroy.png)

***

스트림(Streams)은 [earliest days of unix](http://www.youtube.com/watch?v=tc4ROCJYbm0) 로부터 시작됐고 작은 컴포넌트들([do one thing well](http:/로/www.faqs.org/docs/artu/ch01s06.html))의 조합으로 큰 시스템을 구성하는 믿을 수 있는 방법으로 10여 년이 넘는 기간을 통해 입증되었다.
유닉스에서는 스트림을 쉘에서 파이프기호(`|`) 표현할 수 있다. Node 에서는 별도의 설치 없이 built-in 으로 [stream module](http://nodejs.org/docs/latest/api/stream.html) 을 사용 할 수 있으며 또한 사용자공간(user-space)에서 사용할 수 있다.
유닉스와 유사하 게 node 에서는 `.pipe()` 로 표기하며 메시지 처리에 대한 느린 응답을 역압(backpressure)메커니즘을 통해 자유롭게 사용할 수 있다.

스트림은 [reused](http://www.faqs.org/docs/artu/ch01s06.html#id2877537) 될 수 있는 인터페이스로 구현을 제한하기 때문에 관심분리([separate your concerns](http://www.c2.com/cgi/wiki?SeparationOfConcerns))가 가능하다.
또, 입력으로부터 하나의 스트림 출력을 추가할 수 있고 [use libraries](http://npmjs.org) 는 높은 수준의 흐름관리를 위해 추상화할 수 있다.

스트림은 [small-program design](https://michaelochurch.wordpress.com/2012/08/15/what-is-spaghetti-code/) 과 [unix philosophy](http://www.faqs.org/docs/artu/ch01s06.html) 에 있어서 매우 중요한 컴포넌트이다.
하지만 다른 고려되어야 할 개념들도 많이 있다. 이것만 기억하자. [technical debt](http://c2.com/cgi/wiki?TechnicalDebt) 은 유해하므로 빨리 좋은 개념을 찾아야 한다.

![brian kernighan](http://substack.net/images/kernighan.png)

***

# why you should use streams

Node 에서 I/O  는 비동기로 처리된다. 그래서 디스크나 네트워크를 통한 인터렉션에서는 함수에 콜백을 전달해야 한다.
디스크에서 파일을 서비스하는 경우 아래와 같은 코드를 작성할 수도 있다.

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    fs.readFile(__dirname + '/data.txt', function (err, data) {
        res.end(data);
    });
});
server.listen(8000);
```

이 코드는 동작은 하지만 클라이언트로 결과를 보내기 전에 모든 요청을 위해 `data.txt` 파일 전체를 메모리에 로드한다. 만약 `data.txt` 의 크기가 매우 크다면, 프로그램은 많은 메모리를 차지한 체로 동시에 여러 사용자들에게 서비스를 시작하게 될 것이며 접속은 매우 느릴 것이다.

사용자들은 결과를 받으려면 파일이 서버의 메모리에 모두 로드될 때까지 기다려야 하므로 사용자경험은 매우 실망스러울 것이다.

다행이도 `(req, res)` 두 인자 모두 스트림이다. 이것은 `fs.readFile()` 대신 `fs.createReadStream()` 을 사용해서 더 좋게 만들 수 있다는 것을 의미한다.

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.pipe(res);
});
server.listen(8000);
```
`.pipe()` 는 `fs.createReadStream()` 로 부터 `'data'` 와`'end'` 이벤트를 처리한다. 이 코드는 명확할 뿐만 아니라 `data.txt` 파일이 디스크로부터 받아지는 즉시 하나의 chunk 를 클라이언트로 보낼 것이다.

`.pipe()` 를 사용하면 또 다른 이점이 있다. 클라이언트가 매우 느리거나 high-latency 로 연결할 때 역압을 자동으로 처리할 수 있어서 node 메모리에 chunk 들을 불필요하게 로드하지 않아도 된다.

압축을 원한다면 마찬가지로 아래와 같은 스트리밍 모듈들을 사용할 수 있다.

``` js
var http = require('http');
var fs = require('fs');
var oppressor = require('oppressor');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.pipe(oppressor(req)).pipe(res);
});
server.listen(8000);
```

파일은 브라우저에서 지원하는 gzip 또는 deflate 로 형태로 압축된다. 단지 [oppressor](https://github.com/substack/oppressor) 를 사용해 처리하면 된다.

일단 스트림 API 를 배운다면, 불안정한 사용자정의 API 를 통해 데이터를 전달하는 방법을 기억하기보다는 레고블럭 또는 정원의 호스와 같은 스트리밍 모듈들만 신경쓰면 된다.

스트림은 node 프로그래밍을 간단하고 우아하고 안정적으로 만들 수 있다.

# basics

다음과 같이 5가지 스트림이 있다: readable, writable, transform, duplex, classic.

## pipe

모든 스트림들은 입력과 출력 쌍으로 `.pipe()` 를 사용한다.

`.pipe()` 는 입력(readable) 스트림 `src` 읽어 들이고 출력(writable) 스트림 `dst` 로 출력을 가로채는 함수다:

```
src.pipe(dst)
```

`.pipe(dst)` 는 `dst` 를 반환한다. 또 다수의 `.pipe()` 를 체인으로 호출할 수 있다:

``` js
a.pipe(b).pipe(c).pipe(d)
```

이것은 아래 코드와 동일하게 동작한다:

``` js
a.pipe(b);
b.pipe(c);
c.pipe(d);
```