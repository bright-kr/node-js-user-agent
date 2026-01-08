# Node.js에서 User Agent 설정 및 변경

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.co.kr/) 

이 가이드는 Node.js로 `User-Agent` ヘッダー를 설정하는 방법과 [scraping with node.js](https://brightdata.co.kr/blog/how-tos/web-scraping-with-node-js) 중 アンチボット 탐지를 회피하기 위해 user agent rotation을 구현하는 방법을 설명합니다:

- [Fetch API를 사용하여 Node.js User Agent를 변경하는 방법](#how-to-change-the-nodejs-user-agent-using-the-fetch-api)
  - [로컬로 User Agent 설정](#set-a-user-agent-locally)
  - [전역으로 User Agent 설정](#set-a-user-agent-globally)
- [Node.js에서 User Agent Rotation 구현](#implement-user-agent-rotation-in-nodejs)
  - [Step 1: User Agents 목록 가져오기](#step-1-retrieve-a-list-of-user-agents)
  - [Step 2: 무작위로 User Agent 선택](#step-2-randomly-pick-a-user-agent)
  - [Step 3: 무작위 User Agent로 HTTP リクエスト 수행](#step-3-make-the-http-request-with-a-random-user-agent)
  - [Step 4: 모두 합치기](#step-4-put-it-all-together)

## User Agent 설정이 중요한 이유

[`User-Agent`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent) ヘッダー는 HTTP リクエスト를 수행하는 클라이언트를 식별하는 문자열입니다. 일반적으로 브라우저, 애플리케이션, 운영체제, 시스템 아키텍처에 대한 세부 정보를 포함합니다.

예를 들어, Chrome이 リクエスト를 만들 때 설정하는 user agent 문자열은 다음과 같습니다:

```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36
```

아래는 이 user agent 문자열의 구성 요소를 분석한 내용입니다:

* `Mozilla/5.0`: 원래 Mozilla 브라우저와의 호환성을 나타내기 위해 사용되었으며, 현재는 호환성 목적으로 이 접두사가 추가됩니다.
* `Windows NT 10.0; Win64; x64:` 운영체제(`Windows NT 10.0`), 플랫폼(`Win64`), 시스템 아키텍처(`x64`)를 나타냅니다.
* `AppleWebKit/537.36`: Chrome이 사용하는 브라우저 엔진을 의미합니다.
* (`KHTML, like Gecko`): KHTML 및 Gecko 레이아웃 엔진과의 호환성을 보여줍니다.
* `Chrome/127.0.0.0`: 브라우저 이름과 버전을 지정합니다.
* `Safari/537.36`: Safari와의 호환성을 나타냅니다.

`User-Agent` ヘッダー는 リクエスト가 신뢰할 수 있는 브라우저에서 왔는지 또는 자동화된 소프트웨어에서 왔는지 판단하는 데 도움이 됩니다.

Webスクレイピング 봇은 종종 기본값 또는 비브라우저 user agents를 사용하므로 アンチボット 시스템의 쉬운 표적이 됩니다. 이러한 시스템은 `User-Agent` ヘッダー를 분석하여 실제 사용자와 봇을 구분합니다. 자세한 내용은 [user agents for web scraping](https://brightdata.co.kr/blog/how-tos/user-agents-for-web-scraping-101)에서 확인하시기 바랍니다.

## Node.js 기본 User Agent란 무엇입니까?

버전 18부터 Node.js는 [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)의 내장 구현으로 [`fetch()`](https://nodejs.org/dist/latest/docs/api/globals.html)를 포함합니다. 이는 외부 의존성 없이 Node.js에서 HTTP リクエスト를 수행하는 권장 방식입니다. 자세한 내용은 [HTTP requests in Node.js with Fetch API](/blog/how-tos/fetch-api-nodejs) 가이드를 참고하시기 바랍니다.

대부분의 HTTP 클라이언트와 마찬가지로 `fetch()`는 기본 `User-Agent` ヘッダー를 자동으로 설정합니다. 동일한 동작은 [Python `requests` library](/faqs/python-requests/what-is-python-requests)에서도 발생합니다.

기본적으로 Node.js의 `fetch()`는 다음 `User-Agent` 문자열을 설정합니다:

```
node
```

`fetch()`가 설정하는 기본 `User-Agent`를 확인하려면 [`httpbin.io/user-agent`](https://httpbin.io/user-agent)로 GET リクエスト를 보내면 됩니다. 이 エンドポイント는 수신 リクエスト의 `User-Agent` ヘッダー를 반환하므로, HTTP 클라이언트가 사용하는 user agent를 식별할 수 있습니다.

이를 테스트하려면 Node.js 스크립트를 만들고, [`async`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) 함수를 정의한 다음 `fetch()`로 リクエスト를 수행합니다:

```js
async function getFetchDefaultUserAgent() {

// make an HTTP request to the HTTPBin endpoint

// to get the user agent

const response = await fetch("https://httpbin.io/user-agent");

// read the default user agent from the response

// and print it

const data = await response.json();

console.log(data);

}

getFetchDefaultUserAgent();
```

위 JavaScript 코드를 실행하면 다음 문자열을 받게 됩니다:

```
{ 'user-agent': 'node' }
```

기본적으로 Node.js의 `fetch()`는 `User-Agent`를 `node`로 설정하며, 이는 브라우저 user agents와 크게 다릅니다. 이는 [anti-bot systems](/webinar/bot-detection)을 트리거할 수 있습니다.

アンチボット 솔루션은 비정상적인 user agents를 감지하고 이러한 リクエスト를 봇으로 플래그 처리하여 차단으로 이어질 수 있습니다. 기본 Node.js `User-Agent`를 변경하면 탐지를 피하는 데 도움이 됩니다.

## Fetch API를 사용하여 Node.js User Agent를 변경하는 방법

Fetch API 사양에는 `User-Agent`를 변경하기 위한 내장 메서드가 포함되어 있지 않습니다. 그러나 이는 단지 HTTP ヘッダー이므로 [`fetch()` ヘッダー options](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#setting_headers)를 사용하여 값을 커스터마이즈할 수 있습니다.

**로컬로 User Agent 설정**

`fetch()`는 `headers` 옵션을 통해 ヘッダー 커스터마이징을 지원합니다. 이를 사용하여 특정 HTTP リクエスト를 수행할 때 `User-Agent` ヘッダー를 다음과 같이 설정합니다:

```js
const response = await fetch("https://httpbin.io/user-agent", {

headers: {

"User-Agent":

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

},

});
```

이를 모두 합치면 다음과 같습니다:

```js
async function getFetchUserAgent() {

// make an HTTP request to HTTPBin

// with a custom user agent

const response = await fetch("https://httpbin.io/user-agent", {

headers: {

"User-Agent":

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

},

});

// read the default user agent from the response

// and print it

const data = await response.json();

console.log(data);

}

getFetchUserAgent();
```

위 스크립트를 실행하면 이번에는 결과가 다음과 같습니다:

```
{

'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36'

}
```

**전역으로 User Agent 설정**

リクエスト마다 `User-Agent`를 설정하는 것은 간단하지만 반복적인 코드로 이어질 수 있습니다. 그러나 `fetch()` API는 현재 기본 설정에 대한 전역 오버라이드를 지원하지 않습니다.

이를 우회하기 위해, 원하는 구성으로 `fetch()`를 커스터마이즈하는 wrapper 함수를 만들 수 있습니다:

```js
function customFetch(url, options = {}) {

// custom headers

const customHeaders = {

"User-Agent":

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

...options.headers, // merge with any other headers passed in the options

};

const mergedOptions = {

...options,

headers: customHeaders,

};

return fetch(url, mergedOptions);

}
```

이제 `fetch()` 대신 `customFetch()`를 호출하여 커스텀 user agent로 HTTP リクエスト를 수행할 수 있습니다:

```js
const response = await customFetch("https://httpbin.io/user-agent");
```

전체 Node.js 스크립트는 다음과 같습니다:

```js
function customFetch(url, options = {}) {

// add a custom user agent header

const customHeaders = {

"User-Agent":

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

...options.headers, // merge with any other headers passed in the options

};

const mergedOptions = {

...options,

headers: customHeaders,

};

return fetch(url, mergedOptions);

}

async function getFetchUserAgent() {

// make an HTTP request to HTTPBin

// through the custom fetch wrapper

const response = await customFetch("https://httpbin.io/user-agent");

// read the default user agent from the response

// and print it

const data = await response.json();

console.log(data);

}

getFetchUserAgent();
```

위 Node.js 스크립트를 실행하면 다음이 출력됩니다:

```
{

'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36'

}
```

## Node.js에서 User Agent Rotation 구현

기본 `User-Agent`를 브라우저 문자열로 단순히 대체하는 것만으로는 アンチボット 탐지를 우회하지 못할 수 있습니다. 동일한 IP에서 동일한 user agent로 여러 リクエスト가 발생하면, anti-scraping 시스템은 여전히 해당 활동을 자동화로 플래그 처리할 수 있습니다.

Node.js에서 탐지 위험을 줄이려면 リクエスト에 변동성을 도입하시기 바랍니다. 효과적인 방법 중 하나는 **user agent rotation**이며, 이는 각 リクエスト마다 `User-Agent` ヘッダー가 변경되는 방식입니다. 이를 통해 リクエスト가 서로 다른 브라우저에서 오는 것처럼 보이게 하여 차단될 가능성을 줄입니다.

이제 Node.js에서 user agent rotation을 구현해 보겠습니다.

### Step #1: User Agents 목록 가져오기

[WhatIsMyBrowser.com](https://www.whatismybrowser.com/guides/the-latest-user-agent/) 같은 사이트를 방문하여 유효한 user agent 값 몇 가지로 목록을 채우시기 바랍니다:

```js
const userAgents = [

"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

"Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:128.0) Gecko/20100101 Firefox/128.0",

"Mozilla/5.0 (Macintosh; Intel Mac OS X 14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.5 Safari/605.1.15",

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36 Edg/126.0.2592.113",

// other user agents...

];
```

> **Tip**:
> 
> 이 배열에 포함된 실제 환경의 user agent 문자열이 많을수록 アンチボット 탐지를 피하는 데 더 유리합니다.

### Step #2: 무작위로 User Agent 선택

목록에서 user agent 문자열을 무작위로 선택하여 반환하는 함수를 만듭니다:

```js
function getRandomUserAgent() {

const userAgents = [

// user agents omitted for brevity...

];

// return a user agent randomly

// extracted from the list

return userAgents[Math.floor(Math.random() * userAgents.length)];

}
```

이 함수에서 일어나는 일을 정리하면 다음과 같습니다:

* [`Math.random()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random)는 0과 1 사이의 난수를 생성합니다.
* 그런 다음 이 숫자에 `userAgents` 배열의 길이를 곱합니다.
* [`Math.floor()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/floor)는 그 결과를 해당 숫자 이하의 가장 큰 정수로 내림합니다.
* 앞선 연산의 결과 숫자는 0부터 `userAgents.length - 1`까지의 무작위 인덱스에 해당합니다.
* 마지막으로 그 인덱스를 사용해 user agents 배열에서 무작위 user agent를 반환합니다.

`getRandomUserAgent()` 함수를 호출할 때마다, 대부분 다른 user agent를 얻게 됩니다.

### Step #3: 무작위 User Agent로 HTTP リクエスト 수행

`fetch()`를 사용하여 Node.js에서 user agent rotation을 구현하려면, `getRandomUserAgent()` 함수의 값으로 `User-Agent` ヘッダー를 설정합니다:

```js
const response = await fetch("https://httpbin.io/user-agent", {

headers: {

"User-Agent": getRandomUserAgent(),

},

});
```

이제 Fetch API를 통해 수행되는 HTTP リクエスト는 무작위 user agent를 갖게 됩니다.

### Step #4: 모두 합치기

이전 단계의 스니펫을 Node.js 스크립트에 추가한 다음, `fetch()` リクエスト를 수행하는 로직을 `async` 함수로 감싸시기 바랍니다.

최종 Node.js user agent rotation 스크립트는 다음과 같이 구성됩니다:

```js
function getRandomUserAgent() {

const userAgents = [

"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36",

"Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:128.0) Gecko/20100101 Firefox/128.0",

"Mozilla/5.0 (Macintosh; Intel Mac OS X 14_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.5 Safari/605.1.15",

"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36 Edg/126.0.2592.113",

// other user agents...

];

// return a user agent randomly

// extracted from the list

return userAgents[Math.floor(Math.random() * userAgents.length)];

}

async function getFetchUserAgent() {

// make an HTTP request with a random user agent

const response = await fetch("https://httpbin.io/user-agent", {

headers: {

"User-Agent": getRandomUserAgent(),

},

});

// read the default user agent from the response

// and print it

const data = await response.json();

console.log(data);

}

getFetchUserAgent();
```

스크립트를 3~4번 실행해 보시기 바랍니다. 통계적으로 아래와 같이 서로 다른 user agent リスポンス를 확인할 수 있어야 합니다:

![different user agent responses](https://github.com/luminati-io/node-js-user-agent/blob/main/images/different-user-agent-responses-1024x298.png)

이는 user agent rotation이 효과적으로 작동하고 있음을 보여줍니다.

Et voilà! 이제 Fetch API를 사용하여 Node.js에서 user agents를 설정하는 데 능숙해지셨습니다.

## Conclusion

Node.js에서 user agent rotation을 구현하면 기본적인 anti-scraping 시스템을 회피하는 데 도움이 됩니다. 그러나 더 고급 시스템은 여전히 자동화된 リクエスト를 감지하고 차단할 수 있습니다. IP ban을 피하려면 IP 및 user agent rotation과 같은 기능을 통해 anti-scraping 조치를 효과적으로 우회하여 Webスクレイピング을 그 어느 때보다 쉽게 만드는 [Web Scraper API](https://brightdata.co.kr/products/web-scraper)를 고려하시기 바랍니다.

지금 가입하고 오늘 무료 체험을 시작해 보시기 바랍니다!