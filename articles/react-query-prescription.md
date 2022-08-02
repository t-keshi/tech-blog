---
title: "ReactQueryの処方箋"
emoji: "💊"
type: "tech"
topics: ["react", "typescript", "reactquery"]
published: true
---

# この記事

本記事は、下記のReactQuery公式ドキュメントの内容をベースに、自分なりに噛み砕いてまとめたものになります。

ReactQuery公式ドキュメント
https://tanstack.com/query/v4

コードの大部分は公式サイトから引用してきていますので、その点はご容赦ください。
また、わかりにくい点などがありましたら、お気軽にコメントいただけると嬉しいです🙏

※`ReactQuery`は`Solid`や`Vue`、`Svelte`への対応を進めており、現在の正式な名称は`TanStackQuery`になっています。
`ReactQuery`の方が耳馴染みのある方も多いため、この記事では、`ReactQuery`と呼び、`React`での採用を前提として書いています。

# 対象

できるだけ、Reactにあまり詳しくない人にもわかるように書いていきます。
「データフェッチのカスタムフックくらい自作できる」という玄人の方は、
先に[setLoading、駆逐してやる!!この世から...一匹残らず...!【進撃のデータフェッチ】](https://zenn.dev/t_keshi/articles/attack-on-react-query)をお読みいただけると、
「大人しくライブラリを導入する方が賢明だ」という結論に至るかもしれません。

# ReactQuery is 何？

ReactQueryは、APIとのやりとりを極限まで楽にしてくれるライブラリです。

`ReactQuery`を使うと、次のようなメリットがあります。
- APIからデータをフェッチしたり、データをアップデートする際の、記述をシンプルに見通しよく書くことができる
- キャッシュを使って、余計なリクエストを減らしたり、ローディングに関わるユーザー体験を劇的に改善することができる

お手軽に導入できて、ラーニングカーブが非常に緩やかなところが嬉しいポイントです。

## 前置きはいいからコードはよ

`ReactQuery`を使ってAPIとのやりとりをどれくらい簡潔にできるのか理解するには、コードを覗いてみるのが一番です。
まず、`ReactQuery`を使っていない場合、

```js:NonReactQuery.jsx
import { axios } from "axios"

const  NonReactQuery = () => {
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState()
  const [data, setData]= useState()

  useEffect(() => {
    setLoading(true)
    const asyncFn = () => {
      axios.get("https://www.googleapis.com/books/v1/volumes?q=vscode").then(res => {
        setData(data)
        setLoading(false)
      }).catch((err) => {
        setError(err.message)
        setLoading(false)
      })
    }
    asyncFn()
  }, [])

  if(isLoading){
    return "loading..."
  }

   if(error){
    return "error!"
  }

  return  // 省略
}
```

↑だいたいこんな感じになるかと思います。
これをReactQueryを使って書くと、、、↓

```js:WithReactQuery.jsx
import { axios } from "axios"
import { useQuery } from '@tanstack/react-query'

const  WithReactQuery = () => {
  const { isLoading, error, data } = useQuery(['repoData'], () => axios.get('https://api.github.com/repos/tannerlinsley/react-query'))

  if(isLoading){
    return "loading..."
  }

   if(error){
    return "error!"
  }

  return  // 省略
}
```

17行あったデータ取得に関わるコードが、
`const { isLoading, error, data } = useQuery(['repoData'], () => axios.get('https://api.github.com/repos/tannerlinsley/react-query'))`
のわずか1行で簡潔に書けました！

ちなみにこれは最も基本的な使い方であって、`ReactQuery`の機能のほんの一部です。
より発展的な使い方は、順次ご紹介していきます。

# ReactQueryの特徴

- APIから取得してきたデータ状態管理
- 開発者とユーザーの両方のエクスペリエンスを直接改善する
- 開発効率を上げるDeveloperToolsつき
- RESTだけでなく、GraphQLも対応
- SSR、Next.js対応

つまり、こんな方におすすめのライブラリです。

- `redux-thunk`や`redux-saga`を使ったデータ取得にお疲れの方
- 無限に続く非同期処理のスパゲッティに食あたりをおこした方
- 非同期処理をとにかくお手軽に書きたい方
- 開発しやすさとUXのどちらも犠牲にしたくない方
- `React`の宣言的な設計に共感し、データ取得も宣言的にやりたい方

# ReactQueryのはじめかた

セットアップは簡単で、初めてデータが返ってくるまで2分とかかりません。

まずは、プロジェクトのルートディレクトリで

```shell
$ npm i @tanstack/react-query
# or
$ yarn add @tanstack/react-query
```

を叩いて`ReactQuery`をインストールします。

次に、`React`のルートに近い部分、例えば`App.jsx`で、次のように`Provider`を呼び出します。

```js:App.jsx
import { QueryClient, QueryClientProvider, useQuery } from '@tanstack/react-query'

const queryClient = new QueryClient()

export const App = () => {
  return (
    <QueryClientProvider client={queryClient}>
      <Example />
    </QueryClientProvider>
  )
}
```

あとは、先ほどのようにAPIを叩いてあげるだけです！

#  ReactQueryの使い方(初級編)

ここまでで、だいたいどういったライブラリなのか、雰囲気は伝わったかと思います。
ここからは、もう少し具体的に`ReactQuery`の使い方について説明していきます。

実は`ReactQuery`には既出の`useQuery`というデータ取得用の機能以外にも、`useMutation`というデータ更新用の機能があります。
とりあえず、GETに対応しているのが`useQuery`、PUT・POST・DELETEに対応しているのが`useMutation`というイメージで大丈夫です。
以下、`useQuery`、`useMutation`の順に詳しく見ていきます。

## useQueryの使い方

### axiosでもfetchでもOK!

先ほどの例では、`axios`を使いましたが、Web標準の`fetch`でもデータ取得が可能です。[^1]
ただし、`fetch`を使う場合は、「データ取得がうまくいかなかった場合でもエラーにならない」という`fetch`の仕様に注意してください。
`fetch`を使うときは次のように書いて、レスポンスがokでなかったときにエラーを吐くようにしてあげるのがおすすめです。

```js: getRepoData.js
useQuery(['repoData'], async () => {
  const response = await fetch("https://api.github.com/repos/tannerlinsley/react-query", { method: 'GET'})
  if (!response.ok) {
    throw new Error('Failed request: invalid response')
  }
  return response.json()
})
```

## useQueryの使い方

`useQuery`のリファレンスを読むと、色んなoptionがあって戸惑うかもしれませんが、
とりあえずは`onSuccess`, `onError`, `onSettled`, `enabled`だけ理解していれば、一通りのケースに対応できると思います。
また、`useQuery`が返す値は、`data`,  `isLoading`, `isError`の3つだけ覚えておけば一旦は十分です。
ここでは、optionを順に見ていきます。

### useQueryのオプション: onSuccess, onError, onSettled,

- onSuccess: リクエストが成功したときに発火するイベント
- onError: リクエストが失敗したときに発火するイベント
- onSettled: リクエストが成功したときでも失敗したときでも最後に発火するイベント

具体的には、次のような使い方が可能です。

```js:UserProfilePage.jsx
import axios  from "axios";
import { useToast } from "react-toast-notification"
import { useHistory } from "react-router-dom"

const UserProfilePage = () => {
  const { addToast } = useToast()
  const { data } = useQuery(['profileData'], () => axios.get('https://your-api-endpoint/user/profile'), {
    onSuccess: (data) => addToast({ message: `こんにちは、${data.name}さん！`, appearance: "success"})
    onError: (error) => addToast({ message: `データ取得に失敗しました。Error: ${error.message}`, appearance: "error"})
  })

  return <ProfileTemplate data={data} />
}
```

非同期処理に詳しい方は、`onSuccess`, `onError`, `onSettled`がそれぞれ、`then`, `catch`, `finally`に対応していると考えるとわかりやすいかもしれません。
次のようなイメージです。

```js:react-query/useQuery.js
const useQuery = (_, asyncFn, options) => {
  useEffect(() => {
    asyncFn().then(data => {
      options.onSuccess(data)
    }).catch(error => {
      options.onError(error)
    }).finally(() => {
      options.onSettled()
    }
  }, [])
```

### useQueryのオプション: enabled

`useQuery`は、enabledの値が`true`になっているときだけ、リクエストを投げます。
次のようなイメージで、enabledが`true`以外のときは非同期処理が発火しないことを、頭の片隅に置いといてください。

```js:react-query/useQuery.js
const useQuery = (_, asyncFn, options) => {
  useEffect(() => {
    if(options.enabled === true) {
      asyncFn()
    }
  }, [])
```

具体的に`enabled`のユースケースは後ほどご紹介するので、今はピンとこなくても大丈夫です

## useMutationの使い方

`useMutation`も公式のリファレンスを読むと、optionsが多くて、難しそうに感じるかもしれません。
でも、だいたい`useQuery`と一緒なので、ここはさらっと読み流していただくだけでOKです！

### useMutationのオプション: onSuccess, onError, onSettled,

`useMutation`は、`useQuery`と同じように`onSuccess`, `onError`, `onSettled`をオプションとして受け取ることができます。
具体的には、次のような使い方が可能です。

```js:UserInvitePage.jsx
import axios  from "axios";
import { useToast } from "react-toast-notification"
import { useHistory } from "react-router-dom"

const UserInvitePage = () => {
  const { addToast } = useToast()
  const history = useHistory()
  const { isLoading } = useMutation(() => axios.get('https://your-api-endpoint/user/invite'), {
    onSuccess: (data) => {
      addToast({ message: `${data.username}さんに、招待メールを送りました。3秒後にユーザー一覧画面に戻ります。`, appearance: "success"})
      setTimeout(() => history.push('/user-list'), 3000)
    }
  })

  if(isLoading){
    return <p>招待メールを送信中です...</p>
  }

  return <UserInviteForm />
}
```

useMutationのオプションの中では、この3つの他にも`invalidateQueries`が重要なものですが、説明は後ろに回します。
ここまでで、基本的な使い方の説明は終わりです！お疲れさまでした！

#  ReactQueryの使い方(中級編)

ここまででざっくりと基本的な使い方をさらいましたが、実務レベルでちゃんと使いこなすためには、
`ReactQuery`の重要な概念である`Query Key`と`Cache`の仕組みについて知っておかなければいけません。
中級編では、`Query Key`と`Cache`の仕組みについて説明します。

## Query Key is 何？

すでに出てきた、`useQuery`の使用例では、

```js:UserProfilePage.jsx
const { data } = useQuery(['profileData'], () => axios.get('https://your-api-endpoint/user/profile'),
```

というような形で、一つ目の引数で、配列(ここでは['profileData'])を渡していました。
鋭い方は気になっていた方かもしれませんが、これが`Query Key`です。

Query Keyは、大きく分けて次の2つの役割をするキーです。
①キーの値が変わったタイミングで、データを取得するためのキー
②キャッシュとのデータのやりとりを行うためのキー
（②については後ほど説明するので、一旦忘れてください）

`useQuery`は`Query Key`が変わったタイミングでAPIを叩きます。
だから、`Query Key`に正しい値をちゃんと設定してやらないと、データをうまく取得できないことがあります。
検索のページを例にとり、具体的に`Query Key`の使い方を見ていきます。

```js:BookSearchPage.jsx
import React, { useState } from "react"
import { useQuery } from "@tanstack/react-query"

const BookSearchPage = () => {
  const [keyword, setKeyword] = useState()
  const { data, isLoading }  = useQuery(
    ['bookData', keyword],
    () => axios.get(`https://www.googleapis.com/books/v1/volumes?q=search+${keyword}`).then(res => res.data),
    { enabled: keyword !== undefined }
  )

  return (
    <>
      <input placeholder="検索ワードを入力してください" value={keyword} onChange={(e) => setKeyword(e.target.value)} />
      <div>
        <h1>検索ワードにヒットしたデータ：</h1>
        {data && data.items.map((item) => <p>{item.id}</p>)}
      </div>
    </>
  )
}
```

まず先ほど出てきたenabledを使って、`{ enabled: keyword !== undefined }`のように渡してあげているので、
ユーザーが何も検索ワードを入力していないときは、余計なリクエストを投げません。
そして、`Query Key`には`keyword`を渡しあげているので、keywordが変わったタイミングで、リクエストを投げます。
これを、

```js:BookSearchPage.jsx
const { data, isLoading }  = useQuery(
  ['bookData', keyword],
  () => axios.get(`https://www.googleapis.com/books/v1/volumes?q=search+${keyword}`).then(res => res.data),
  { enabled: keyword !== undefined }
)
```

と書いてしまうと、ユーザーが何か検索ワードを入力しても、リクエストは走らず、沈黙のままになるので気をつけてください。
基本的には、リクエストのパラメータ(今回の例で言うと`keyword`のこと)をすべて`Query Key`に、含めるようにするのが良いと思います。

`react`に詳しい方は一旦、次のように`Query Key`は`useEffect`の依存配列のことだと捉えておくとわかりやすいでしょう。
`useEffect`の依存配列に嘘をついてはいけないように、`Query Key`にもできるだけを嘘をつかないようにしてください。

```js:react-query/useQuery.js
const useQuery = (queryKey, asyncFn, options) => {
  useEffect(() => {
    asyncFn()
  }, queryKey)
```

## Cache is 何？

`ReactQuery`のキャッシュがどんなものなのか説明する前に、`Cache`があると何が嬉しいのか説明します。
まず、`Cache`がない場合のユーザー体験です。

![slow](/images/slow.gif)

次に、`Cache`がある場合のユーザー体験です。

![fast](/images/fast.gif)

両者を比較すると、同じAPIを叩いているのに、`Cache`がある場合の方がキビキビ動いているように見えます。
具体的に何が違うのかというと、`Cache`がある例では、一度投げたリクエストは瞬時に結果がデータが取得できるようになっています。

`ReactQuery`は、APIから返ってきたデータをしばらくの間、保持してくれる機能を持っています。
このとき、データを一時的に保持しておく場所を`Cache`といいます。
そして、次に同じリクエストをしたときには、`Cache`に保持していたデータを取り出して使うことができるため、
サーバーにデータを取得するより早いスピードで、ユーザーにデータを見せることができます。
具体的にコードで見ていきます。

```js:UniversitiesPage.jsx
import React, { useState } from "react"
import { useQuery } from "@tanstack/react-query"

const getUniversities= (page: number) =>
  fetch("https://your-api-endpoint/universities?page=" + page).then(res => res.json())

const UniversitiesPage= () => {
  const [page, setPage] = useState(1);
  const { data } = useQuery(
    ["universitiesData", page]
    () => getUniversities(page),
  );

  if(isLoading){
    return <p>loading...</p>
  }

  return (
    <>
      {data && data.map((university) => (
        <p key={university.id}>{university.name}</p>
      ))}
      <div>
        <button onClick={() => setPage(page - 1)}>⇦前へ</button>
        <button onClick={() => setPage(page + 1)}>次へ⇨</button>
      </div>
    </>
  );
};
```

この例では、
①最初に１ページ目に訪れたとき：APIからのレスポンスを待って画面を表示
②「次へ⇨」を押して２ページ目に訪れたとき：APIからのレスポンスを待って画面を表示
③「⇦前へ」２ページ目から１ページに戻ったとき：`Cache`に保持している①のデータを取り出して、瞬時に画面を表示
という挙動になるため、キャッシュの仕組みのないアプリケーションをより動作が軽快な印象を与えることができます。

## Cacheの仕組み

さきほど

> Query Keyは、大きく分けて次の2つの役割をするキーです。
> ①キーの値が変わったタイミングで、データを取得するためのキー
> ②キャッシュとのデータのやりとりを行うためのキー
>（②については後ほど説明するので、一旦忘れてください）

と書いたところの、飛ばした②の部分を説明していきます。

`ReactQuery`は`Cache`と呼ばれるキャビネットの中に、APIから取得したデータを入れたファイルを投げ込んでいきます。
このとき、ファイルに貼るラベルのようなものがないと、せっかく収納したデータも取り出し方がわからなくなってしまいます。
わからなくなるだけならまだましですが、誤って取り出してきてしまうかもしれません。

「ファイルに貼るラベル」の役割をするのが、先ほど出てきた`Query Key`です。
敢えて、`Query Key`に`hoge`といういいかげんな値を設定して、ファイルに貼る正しいラベルを貼っておかないとどうなるかを見てみます。

```js:BookPage.jsx
import React, { useState } from "react"
import { useQuery } from "@tanstack/react-query"
import { Link } from "react-router-dom"

const BookPage = () => {
  const { data }  = useQuery(
    ['hoge'],
    () => axios.get(`https://www.googleapis.com/books/v1/volumes?q=search+`).then(res => res.data),
  )

  return (
    <>
      <div>
        {data && data.items.map((item) => <p key={item.id}>{item.id}</p>)}
      </div>
      <Link to="/universities">
    </>
  )
}
```

```js:UniversitiesPage.jsx
import React, { useState } from "react"
import { useQuery } from "@tanstack/react-query"

const getUniversities= (page: number) =>
  fetch("https://your-api-endpoint/universities?page=" + page).then(res => res.json())

const UniversitiesPage = () => {
  const [page, setPage] = useState(1);
  const { data } = useQuery(
    ['hoge']
    () => getUniversities(page),
  );

  return (
    <>
      {data && data.map((university) => (
        <p key={university.id}>{university.name}</p>
      ))}
    </>
  );
};
```

ここでは、本のデータと大学のデータという全く別々のデータに同じ`['hoge']`というラベルを貼ってしまったために、エラーが起こります。
エラーが起こる具体的な流れは、まずリンクを押下して`BookPage`から`UniversitiesPage`にジャンプしたとき、`UniversitiesPage`の`useQuery`は、`Query Key`に`['hoge']`が渡されているのを見て、「さっきの`BookPage`と同じデータが使える！」と誤った判断をします。
しかし、「さっきの`BookPage`と同じデータが使える！」と言って`ReactQuery`が渡してきたデータは、`UniversitiesPage`では全く使えないデータなので、「これ思ってたデータとちゃうやんけ💢」というエラーになります。(Uncaught TypeError: map is not a function)

JavaScriptに詳しい人は、CacheはただのMap型のグローバル変数だと捉えておくとわかりやすいと思います。
次のようなイメージです。

```js:react-query/useQuery.js
const cache = new Map()

const useQuery = (queryKey, asyncFn, options) => {
  const [data, setData] = useState()
  useEffect(() => {
    const cachedData = cache.get(JSON.stringify(queryKey))
    if(cachedData){
      setData(cachedData)
    }
    asyncFn().then(data => {
      cache.set(JSON.stringify(queryKey), data)
    })
  }, queryKey)
```

## 古いCacheを無効化する方法