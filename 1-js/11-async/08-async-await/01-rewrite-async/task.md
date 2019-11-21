
# async�� await�� ����Ͽ� �ڵ� �����ϱ�

<info:promise-chaining> é���� ���� �� �ϳ��� `.then/catch` ��� `async/await`�� ����� �ٽ� �ۼ��غ��ô�.

```js run
function loadJson(url) {
  return fetch(url)
    .then(response => {
      if (response.status == 200) {
        return response.json();
      } else {
        throw new Error(response.status);
      }
    })
}

loadJson('no-such-user.json') // (3)
  .catch(alert); // Error: 404
```