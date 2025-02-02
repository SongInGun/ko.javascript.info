# 프라미스 API

'프라미스' 클래스에는 5가지의 정적 메서드가 있습니다. 프라미스의 유스 케이스에 대해서 빠르게 알아보겠습니다.

## Promise.all

여러 개의 프라미스를 동시에 실행시키고 모두 준비가 될 때까지 기다린다는 가정을 해봅시다.

예를 들면 여러 URL을 동시에 다운로드하고 다운로드가 모두 끝나면 내용을 처리하는 것입니다.

그것이 `Promise.all`이 하는 일입니다.

문법은 다음과 같습니다.

```js
let promise = Promise.all([...promises...]);
```

`Promise.all`은 프라미스 배열(엄밀히 따지면 반복 가능한(iterable, 이터러블) 객체나 보통 배열을 말합니다.)을 가져가고 새로운 프라미스를 반환합니다.

나열된 프라미스가 모두 처리되면 새로운 프라미스가 이행되고, 처리된 프라미스의 결괏값을 담은 배열이 새로운 프라미스의 결과가 됩니다.

아래 `Promise.all`은 3초 후에 처리되고, 반환되는 프라미스의 result 값은 배열 `[1, 2, 3]`이 됩니다.

```js run
Promise.all([
  new Promise(resolve => setTimeout(() => resolve(1), 3000)), // 1
  new Promise(resolve => setTimeout(() => resolve(2), 2000)), // 2
  new Promise(resolve => setTimeout(() => resolve(3), 1000))  // 3
]).then(alert); // 모든 프라미스가 준비되면 1, 2, 3이 반환됩니다. 프라미스 각각은 배열을 구성하는 요소가 됩니다.
```

반환되는 배열엔 `Promise.all`에 들어가는 프라미스와 동일한 순서로 요소가 저장된다는 점에 유의하시기 바랍니다. 첫 번째 프라미스가 가장 늦게 이행되더라도 그 결과는 배열의 첫 번째 요소가 됩니다.

작업해야 할 데이터(아래 예시에선 url)가 담긴 배열을 프라미스 배열로 매핑하고, 이 배열을 `Promise.all`로 감싸는 것은 흔히 사용되는 트릭입니다.

아래 예시에선 fetch를 써서 URL이 담긴 배열을 처리합니다.

```js run
let urls = [
  'https://api.github.com/users/iliakan',
  'https://api.github.com/users/remy',
  'https://api.github.com/users/jeresig'
];

// 모든 url에 fetch를 적용해 프라미스로 매핑합니다.
let requests = urls.map(url => fetch(url));

// Promise.all은 모든 작업이 이행될 때까지 기다립니다.
Promise.all(requests)
  .then(responses => responses.forEach(
    response => alert(`${response.url}: ${response.status}`)
  ));
```

좀 더 복잡한 예시를 살펴봅시다. GitHub 유저네임이 담긴 배열을 사용해 사용자 정보를 가져오는 예시입니다(동일한 로직을 사용해 id로 장바구니 목록을 불러올 수 있습니다).

```js run
let names = ['iliakan', 'remy', 'jeresig'];

let requests = names.map(name => fetch(`https://api.github.com/users/${name}`));

Promise.all(requests)
  .then(responses => {
    // 모든 응답이 성공적으로 이행되었습니다.
    for(let response of responses) {
      alert(`${response.url}: ${response.status}`); // 모든 url의 응답코드가 200입니다.
    }

    return responses;
  })
  // 응답 메시지가 담긴 배열을 response.json()로 매핑해, 내용을 읽습니다.
  .then(responses => Promise.all(responses.map(r => r.json())))
  // JSON 형태의 응답 메시지는 파싱 되어 배열 "users"에 저장됩니다.
  .then(users => users.forEach(user => alert(user.name)));
```

**프라미스가 하나라도 거부되면 `Promise.all`이 반환한 프라미스는 에러와 함께 바로 거부됩니다.**

예시:

```js run
Promise.all([
  new Promise((resolve, reject) => setTimeout(() => resolve(1), 1000)),
*!*
  new Promise((resolve, reject) => setTimeout(() => reject(new Error("에러 발생!")), 2000)),
*/!*
  new Promise((resolve, reject) => setTimeout(() => resolve(3), 3000))
]).catch(alert); // Error: 에러 발생!
```

2초 후 두 번째 프라미스가 거부되면 `Promise.all` 전체가 거부되기 때문에 `.catch`가 실행됩니다. 거부된 프라미스의 에러는 `Promise.all` 전체의 결과가 됩니다.

```warn header="에러가 발생하면 다른 프라미스는 무시됩니다."
프라미스가 하나라도 거부되면 `Promise.all`은 즉시 거부되고 배열에 저장된 다른 프라미스의 결과는 완전히 사라집니다. 실패하지 않은 프라미스라도 결과가 무시됩니다.

위 예제처럼 `fetch`를 여러 번 호출하고, 그중 하나가 실패해도 다른 호출은 계속 실행됩니다. 그러나 `Promise.all`은 다른 호출을 더 이상 신경 쓰지 않습니다. 프라미스가 처리되긴 하겠지만 그 결과는 무시됩니다.

프라미스에는 '취소'라는 개념이 없어서 `Promise.all`도 프라미스를 취소하지 않습니다. [또 다른 챕터](info:fetch-abort)에서 배울 `AbortController`를 사용하면 프라미스 취소가 가능하긴 하지만 이는 프라미스 API는 아닙니다.
```

````smart header="`이터러블 객체`가 아닌 '일반' 값도 `Promise.all(iterable)`에 넘길 수 있습니다."
`Promise.all(...)`은 대게 프라미스가 요소인 이러터블 객체(대부분 배열)를 받습니다. 그런데 프라미스가 아닌 객체가 배열을 구성하면, 요소가 '그대로' 결과 배열로 전달됩니다.

아래 예시의 결과는 `[1, 2, 3]`입니다.

```js run
Promise.all([
  new Promise((resolve, reject) => {
    setTimeout(() => resolve(1), 1000)
  }),
  2,
  3
]).then(alert); // 1, 2, 3
```

따라서 이미 결과를 알고 있는 값을 `Promise.all`을 사용해 보낼 수 있습니다.
````

## Promise.allSettled

[recent browser="new"]

`Promise.all`은 프라미스가 하나라도 거절되면 전체를 거절합니다. 따라서, 프라미스 결과가 *모두* 필요할 때같이 "모 아니면 도" 같은 상황에 유용합니다.

```js
Promise.all([
  fetch('/template.html'),
  fetch('/style.css'),
  fetch('/data.json')
]).then(render); // render 메서드는 모든 fetch 메서드의 결괏값이 필요합니다.
```

반면, `Promise.allSettled`는 모든 프라미스가 처리될 때까지 기다립니다. 반환되는 배열은 다음과 같은 요소를 갖습니다.

- 응답이 성공할 경우 -- `{status:"fulfilled", value:result}`
- 에러가 발생한 경우 -- `{status:"rejected", reason:error}`

`fetch`를 사용해 여러 명의 정보를 가져오고 있다고 가정해봅시다. 요청 하나가 실패해도 다른 요청 결과는 여전히 필요합니다.

이럴 때 `Promise.allSettled`를 사용할 수 있습니다.

```js run
let urls = [
  'https://api.github.com/users/iliakan',
  'https://api.github.com/users/remy',
  'https://no-such-url'
];

Promise.allSettled(urls.map(url => fetch(url)))
  .then(results => { // (*)
    results.forEach((result, num) => {
      if (result.status == "fulfilled") {
        alert(`${urls[num]}: ${result.value.status}`);
      }
      if (result.status == "rejected") {
        alert(`${urls[num]}: ${result.reason}`);
      }
    });
  });
```

`(*)`로 표시한 줄의 `results`는 다음과 같을 겁니다.
```js
[
  {status: 'fulfilled', value: ...response...},
  {status: 'fulfilled', value: ...response...},
  {status: 'rejected', reason: ...error object...}
]
```

`Promise.allSettled`를 사용하면 이처럼 각 프라미스의 상태와 `값 또는 에러`를 받을 수 있습니다.

### 폴리필

브라우저가 `Promise.allSettled`를 지원하지 않는다면 폴리필을 구현하면 됩니다.

```js
if(!Promise.allSettled) {
  Promise.allSettled = function(promises) {
    return Promise.all(promises.map(p => Promise.resolve(p).then(value => ({
      state: 'fulfilled',
      value
    }), reason => ({
      state: 'rejected',
      reason
    }))));
  };
}
```

위 코드에서 `promises.map`은 입력값을 받아  `p => Promise.resolve(p)`를 사용해 이를 프라미스로 변경합니다(프라미스가 아닌 값을 받은 경우). 그리고 모든 프라미스에 `.then` 핸들러를 덧붙입니다.

`then` 핸들러는 성공한 프라미스의 결괏값 `v`를 `{state:'fulfilled', value:v}`로, 실패한 프라미스의 결괏값 `r`을 `{state:'rejected', reason:r}`로 변경합니다. `Promise.allSettled`의 구성방식과 동일하게 말이죠.

이렇게 폴리필을 구현하면 몇몇 프라미스가 거부되더라도 `Promise.allSettled`를 사용해 주어진 프라미스 *전체*의 결과를 얻을 수 있습니다.

## Promise.race

`Promise.race`는 `Promise.all`와 비슷합니다. 다만 가장 먼저 처리되는 프라미스의 결과(혹은 에러)를 반환합니다.

문법은 다음과 같습니다.

```js
let promise = Promise.race(iterable);
```

아래 예시의 결과는 `1`입니다.

```js run
Promise.race([
  new Promise((resolve, reject) => setTimeout(() => resolve(1), 1000)),
  new Promise((resolve, reject) => setTimeout(() => reject(new Error("에러 발생!")), 2000)),
  new Promise((resolve, reject) => setTimeout(() => resolve(3), 3000))
]).then(alert); // 1
```

첫 번째 프라미스가 가장 빨리 처리상태가 되기 때문에 이 프라미스의 처리 결과가 result 값이 됩니다. "경주(race)의 승자"가 나타나면 다른 프라미스의 결과 또는 에러는 무시됩니다.


## Promise.resolve/reject

프라미스 메서드 `Promise.resolve`와 `Promise.reject`는 `async/await` 문법([뒤에서](info:async-await) 다룸)이 생긴 후로 쓸모없어졌기 때문에 근래에는 거의 사용하지 않습니다.

여기선 튜토리얼의 완성도를 높이고 어떤 이유 때문이라도 `async/await`를 사용하지 못하는 분들을 위해 이 메서드들에 대해서 다루겠습니다.

- `Promise.resolve(value)`는 결괏값이 `value`인 이행된 프라미스를 생성합니다.

아래 코드와 동일한 일을 수행합니다.

```js
let promise = new Promise(resolve => resolve(value));
```

호환성을 유지하면서 프라미스를 반환하는 함수를 구현할 때 `Promise.resolve`를 사용할 수 있습니다.

아래 함수 `loadCached`는 인수로 받은 URL에 `fetch`를 호출하고, 그 결과를 기억(cache)합니다. 나중에 동일한 URL을 대상으로 `fetch`를 호출하면 캐시에서 즉시 저장된 내용을 가져옵니다. 그런데 이때 `Promise.resolve`를 사용하면 반환 값이 항상 프라미스가 되게 할 수 있습니다.

```js
let cache = new Map();

function loadCached(url) {
  if (cache.has(url)) {
*!*
    return Promise.resolve(cache.get(url)); // (*)
*/!*
  }

  return fetch(url)
    .then(response => response.text())
    .then(text => {
      cache.set(url,text);
      return text;
    });
}
```

함수를 호출하면 항상 프라미스가 반환되기 때문에 `loadCached(url).then(…)`을 사용할 수 있습니다. `loadCached` 뒤에 항상 `.then`을 쓸 수 있죠. `(*)`로 표시한 줄에서 `Promise.resolve`를 사용한 이유가 바로 여기에 있습니다.

### Promise.reject

- `Promise.reject(error)`는 결괏값이 `error`인 거부된 프라미스를 생성합니다.

아래 코드와 동일한 일을 수행합니다.

```js
let promise = new Promise((resolve, reject) => reject(error));
```

실무에서 이 메서드를 쓸 일은 거의 없습니다.

## 요약

`Promise` 클래스에는 5가지 정적 메서드가 있습니다.

1. `Promise.all(promises)` -- 모든 프라미스가 이행될 때까지 기다렸다가 그 결괏값을 담은 배열을 반환합니다. 주어진 프라미스 중 하나라도 실패하면 `Promise.all`는 거부되고, 나머지 프라미스의 결과는 무시됩니다.
2. `Promise.allSettled(promises)`(최근에 추가된 메서드) -- 모든 프라미스가 처리될 때까지 기다렸다가 그 결과(객체)를 담은 배열을 반환합니다. 객체엔 다음과 같은 정보가 담깁니다.
    - `state`: `"fulfilled"` 또는 `"rejected"`
    - 프라미스가 성공한 경우엔 `value`, 실패한 경우엔 `reason`
3. `Promise.race(promises)` -- 가장 먼저 처리된 프라미스의 결과 또는 에러를 담은 프라미스를 반환합니다.
4. `Promise.resolve(value)` -- 주어진 값을 사용해 이행 상태의 프라미스를 만듭니다.
5. `Promise.reject(error)` -- 주어진 에러를 사용해 거부 상태의 프라미스를 만듭니다.

실무에선 지금까지 다룬 다섯 가지 메서드 중에서 `Promise.all`을 가장 많이 사용합니다.