# 第6回資料

## 非同期処理

### 非同期処理とは

非同期処理とは、時間がかかる処理が来た際に、その作業が終わるのを待つ（＝同期）のではなく、完了したら後から連絡してもらうこと。

現在、見かける非同期処理の書き方は大きく3種類。

1. コールバック

1. Promise

1. async / await

このうち`コールバック`は、コールバック地獄を避けるため、使用しない方が良い。

ということで、 `Promise` と `async/await` について解説する。

### Promise

```ts
fetch(url).then(resp => {
  return resp.json();
}).then(json => {
  console.log(json);
}).catch(e => {
  // エラー発生時にここを通過する
}).finally(() => {
  // エラーが発生しても、正常終了時もここを通過する
});
```

- then() 節の中に、前の処理が終わった時に呼び出して欲しいコードを書く
- then() の中で return で返されたものが次の then() の入力になる
- then() の中で Promise を返すと、その返された Promise が解決すると、その結果が次の then() の入力になる
- catch() と finally() は通常の例外処理と同じ

### async/await

```ts
const resp = await fetch(url);
const json = await resp.json();
console.log(json);
```

- await は、 Promise を使ったコードの、 then() 節の中だけを並べたのとほぼ等価。
- await を扱うには、 async をつけて定義された関数でなければならない。

```ts
async function(): Promise<number> {
  await 時間のかかる処理();
  return 10;
}
```

- TypeScriptでは、 async を返す関数の返り値は必ず Promise になる。
- ジェネリクスのパラメータとして、返り値の型を設定する。
- Promise を返す関数は、関数の宣言文を見たときに動作が理解しやすくなるので async をつけておく方が良い。

比較的新しく作られたライブラリなどは最初からPromiseを返す実装になっていると思いますが、そうでないコールバック関数方式のコードを扱う時は new Promiseを使ってPromise化する。

```ts
const sleep = async (time: number): Promise<number> => {
  return new Promise<number>(resolve => {
    setTimeout(()=> {
      resolve(time);
    }, time);
  });
};

await sleep(100);
```

末尾がコールバック、コールバックの先頭の引数はErrorという、2010年代の行儀の良いAPIであれば、Promise化してくれるライブラリがある。

```ts
// Node.js標準ライブラリのpromisifyを使う

import { promisify } from "util";
import { readFile } from "fs";
const readFileAsync = promisify(readFile);

const content = await readFileAsync("package.json", "utf8");
```

### 非同期と制御構文

Promise やコールバックを使ったコードで、条件によって非同期処理を1つ追加する、というコードを書くのは大変。一方、await は条件が複雑なケースでも簡単に非同期を含むコードを扱える。

```ts
// たまに（＝2で割った余りが1の時に）実行される
async function randomRun() {
}

// 必ず実行される
async function finallyFunc() {
}

async function main(){
  if (Date.now() % 2 === 1) {
    await randomRun();
  }
  await finallyFunc();
}

main();
```

---

await を使うと、ループを一回回るたびに重い処理が完了するのを待つことができる。

しかし、同じループでも、配列の forEach() を使うと、1要素ごとに await で待つことはできないし、すべてのループの処理が終わったあとに、何かを行わせることもできない。（await と forEach は相性が悪い？）

```ts
// for of, if, while, switchはawaitとの相性も良い
for (const value of iterable) {
  await doSomething(value);
}
console.log("この行は全部のループが終わったら実行される");
```

```ts
// このawaitでは待たずにループが終わってしまう
iterable.forEach(async value => {
  await doSomething(value);
});
console.log("この行はループ内の各処理が回る前に即座に実行される");
```

### Promise.all()

```ts
async function 味噌汁温め(): Promise<味噌汁> {
  await ガスレンジ();
  return new 味噌汁();
}

async function ご飯温め(): Promise<ご飯> {
  await 電子レンジ();
  return new ご飯();
}

const [a味噌汁, aご飯] = await Promise.all([味噌汁温め(), ご飯温め()]);
いただきます(a味噌汁, aご飯);
```

- Promise.all() は、 Promise の配列を受け取り、全部の Promise が完了するのを待つ。
- Promise.all() は、引数のすべての結果が得られると、解決して結果をリストで返す Promise を返す。
- Promise.all() の結果を await すると、すべての結果がまとめて得られる。
- （補足）Promise.all() の引数の配列に、 Promise 以外の要素があると、即座に完了する Promise として扱われる。

上のコードで、

- await をつけると、待った後の結果（ここでは味噌汁とご飯のインスタンス）が返ってくる。
- await をつけないと、 Promise そのものが返ってくる。

類似の関数で Promise.race() というものがある。これは all() と似ているが、全部で揃うと実行されるわけではなく、どれか一つでも完了すると呼ばれる。

#### ループの中の await

ループの中に await を書くと、この await を内部にもつループがボトルネックとなり、ユーザーレスポンスが遅れることもありえる。

この場合、 Promise.all() を使うと、全部の重い処理を同時に投げ、一番遅い最後の処理が終わるまで待つことができる。

```ts
for (const value of iterable) {
  await doSomething(value);
}
```

```ts
await Promise.all(
  iterable.map(
    async (value) => doSomething(value)
  )
);
```

しかし、 Promise.all() が適切ではない場面もいくつかある。

例えば、外部のAPI呼び出しをする場合、たいてい、秒間あたりのアクセス数が制限されているので、100並列でリクエストを投げるとエラーが帰って来て正常に処理が終了しないこともありえる。（その場合は、並列数を制御しつつ map() と同等のことを実現してくれる [p-map](https://www.npmjs.com/package/p-map) といったライブラリを活用すると良い。）

### 練習問題

以下のファイル書き込み処理を、Promise のみを用いたり、 async/await を組み合わせて、非同期 I/O のものから同期 I/O のものに変更してみよう。

```js
'use strict';
const fs = require('fs');
const fileName = './test.txt';
for (let count = 0; count < 30; count++) {
  fs.appendFile(fileName, 'おはようございます\n', 'utf8', () => {});
  fs.appendFile(fileName, 'こんにちは\n', 'utf8', () => {});
  fs.appendFile(fileName, 'こんばんは\n', 'utf8', () => {});
}
```

<!-- ### 練習問題の答え

#### Promise を使った例

```js
'use strict';
const fs = require('fs');
const fileName = './test.txt';
function appendFilePromise(fileName, str) {
  return new Promise((resolve) => {
    fs.appendFile(fileName, str, 'utf8', () => resolve());
  });
}
function main() {
  let promiseChain = Promise.resolve(); // Promise チェーンを記憶する変数
  for (let count = 0; count < 500; count++) {
    promiseChain = promiseChain
      .then(() => {
        return appendFilePromise(fileName, 'おはようございます\n');
      })
      .then(() => {
        return appendFilePromise(fileName, 'こんにちは\n');
      })
      .then(() => {
        return appendFilePromise(fileName, 'こんばんは\n');
      });
  }
}
main();
```

#### async/await を使った例

```js
'use strict';
const fs = require('fs');
const fileName = './test.txt';
function appendFilePromise(fileName, str) {
  return new Promise((resolve) => {
    fs.appendFile(fileName, str, 'utf8', () => resolve());
  });
}
async function main() {
  for (let count = 0; count < 500; count++) {
    await appendFilePromise(fileName, 'おはようございます\n');
    await appendFilePromise(fileName, 'こんにちは\n');
    await appendFilePromise(fileName, 'こんばんは\n');
  }
}
main();
``` -->
