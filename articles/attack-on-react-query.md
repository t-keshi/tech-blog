---
title: 'setLoading、駆逐してやる!!この世から...一匹残らず...!【進撃のデータフェッチ】'
emoji: '🦧'
type: 'tech'
topics: [react, swr, reactquery, typescript, frontend]
published: true
---

今回は、APIとのやりとりを如何にシンプルに、見通しよくするかについて書いていきます。
進撃の巨人の内容がうろ覚えなので、それも含めて間違っている箇所などありましたら、ご指摘のほどよろしくお願いします🙏

書き出しはやはり、これですか、、

## 「その日、人類は思い出したヤツらに支配されていた恐怖を...」

reactにおけるAPIとのやりとりはかつて、`redux-thunk`とか`redux-saga`を使って行うのが一般的でした。

ここでちょっと懐かしく思い出してみます。
`thunk`や`saga`に支配されていた恐怖を...鳥籠の中に囚われていた屈辱を……

actionの肥大化、yieldの黒魔術...。
なかなか大変でしたね...。

でも数年前から、「Reduxってほとんどのケースでオーバーキルだよね？」という話になり、Redux一辺倒の状況に変化の兆しが現れました。次第に、Reduxを使っていないプロダクトを目にすることが増えてきました。

そして、脱Redux化の動きに伴い、データ取得の方法も方法も大きく変わってきています。

`thunk`や`saga`の時代は、それはそれで辛みが大きかったものですが、一応「統一されたルール」が存在していました。けれども、そうしたライブラリなく、`useEffect`一本で書くと、これもまた自由すぎて煩雑になってしまうパターンが多いなぁと感じています。

...まるで、巨人の侵入を防げないウォールローゼの壁のように。

例えば最近、次のようなコードに出会いました。

```ts: StudentSubmitForm.tsx
type University = {
  code: number
  name: string
  isDefault: boolean
}

const StudentSubmitForm: React.FC = () => {
  const [loading, setLoading] = useState<boolean>(false)
  const [error, setError] = useState<Error | undefined>()
  const [options, setOptions] = useState<{ universityCode: number, universityName: string}>({ universityCode: 1, universityName: ""})
  const [universityCode, setUniversityCode]= useState<number | undefined>()

  useEffect(() => {
    setLoading(true)
    const asyncFn = async() => {
        const res =  await fetch("/api/university/list", { method: "GET"})
        const data: University[] = await res.json()
        const optionsData = res.map(({ code, name}) => ({ universityCode: code, universityName: name }))
        setOptions(optionsData)
        const defaultUniversityCode = res.find((uni) => uni.isDefault).code
        setUniversityCode(defaultUniversityCode)
        setLoading(false)
      }).catch((err: Error) => {
        setError(err.message)
        setLoading(false)
      })
    }.
    asyncFn()
  }, [])

  return (
    <Form onSubmit={() => {}}>
      <Dropdown options={options} onSelect={(newCode) => setUniversityCode(newCode)}>
    </Form>
  )
}
```

学生が就活Webサイトに個人情報を登録するときの画面をイメージしてください。
上期のコードは、APIを叩いて、大学の一覧を取得した後、フォームに初期値を入れています。

どっからどこまでAPIとのやりとりの話で、どっからどこまでがフォームの話か、ちょっとわかりづらい気がします。

もう一回、同じコードを見てみます。

```ts: StudentSubmitForm.tsx
const StudentSubmitForm: React.FC = () => {
  // この辺はAPIとのやりとりの話
  const [loading, setLoading] = useState<boolean>(false)
  const [error, setError] = useState<Error | undefined>()

  // この辺はフォームの話
  const [options, setOptions] = useState<{ universityCode: number, universityName: string}>({ universityCode: 1, universityName: ""})
  const [universityCode, setUniversityCode]= useState<number | undefined>()

  useEffect(() => {
    setLoading(true)
    const asyncFn = async() => {
        // この辺はAPIとのやりとりの話
        const res = await fetch("/api/university/list", { method: "GET"}).then((res) => {
        const data: University[] = await res.json()

        // この辺はフォームの話
        const optionsData = data.map(({ universityCode, universityName}) => ({ universityCode, universityName }))
        setOptions(optionsData)
        const defaultUniversityCode = res.find((uni) => uni.isDefault).code
        setUniversityCode(defaultUniversityCode)

        // この辺はAPIとのやりとりの話
        setLoading(false)
      }).catch((err: Error) => {
        // この辺はAPIとのやりとりの話
        setError(err.message)
        setLoading(false)
      })
    }
    asyncFn()
  }, [])

   if(loading){
    return <Loader />
  }

  return (
    // この辺はフォームの話
    <Form onSubmit={() => {}}>
      <Dropdown options={options} onSelect={(newCode) => setUniversityCode(newCode)}>
    </Form>
  )
}
```

やはり思うに、APIとのやりとりの話とフォームの話が混在しているのが問題です。機能追加の要望があって、フォームの入力項目が増えてきたら、いよいよ崩壊しそうな懸念があります。

APIから取得してきたデータはAPIから取得してきたデータとして、そのままStateに入れてあげて、それを使ってフォーム部分を作る話は、また別のところに書いてあげてはどうでしょうか？

実際にやってみます。

```diff ts: StudentSubmitPage.tsx
// APIとのやりとりの話
const StudentSubmitPage: React.FC = () => {
  const [loading, setLoading] = useState<boolean>(false)
  const [error, setError] = useState<Error | undefined>()
  const [universityData, setUniversityData]= useState<University[] | undefined>()

  useEffect(() => {
    setLoading(true)
    const asyncFn = async() => {
      await fetch("/api/university/list", { method: "GET"}).then((res) => {
        const res = await res.json()
- const optionsData = data.map(({ universityCode, universityName}) => ({ universityCode, universityName }))
- setOptions(optionsData)
- const defaultUniversityCode = res.find((uni) => uni.isDefault).code
- setUniversityCode(defaultUniversityCode)
        setUniversityData(res)
        setLoading(false)
      }).catch((err: Error) => {
        setError(err.message)
        setLoading(false)
      })
    }
    asyncFn()
  }, [])

  if(loading){
    return <Loader />
  }

  return (
    <StudentForm universityData={universityData}>
  )
}
```

```diff ts: StudentSubmitForm.tsx
// フォームの話
const StudentSubmitForm: React.FC<{universityData: University[]}> = ({ universityData}) => {
+ const options = universityData.map(({universityCode, universityName}) => ({ universityCode, universityName}))
+ const defaultUniversityCode = universityData.find((university) => university.isDefault).code
+ const [universityCode, setUniversityCode]= useState<number>(defaultUniversityCode)

  return (
    <Form onSubmit={() => {}}>
      <Dropdown options={options} onSelect={(newCode) => setUniversityCode(newCode)}>
    </Form>
  )
}
```

コンポーネントは増えたけど、APIとのやりとりの話とフォームの話の境目がハッキリしました。渾然一体となっていたものが、それぞれの関心に従い、２つに分離したようです。

取得してきたデータをごにょごにょしてから`state`に入れるのではなく、取得してきたデータをそのまま`state`に突っ込んだ上でそれを加工するように変えるだけで、コードの責務がハッキリします。パフォーマンスの観点からは、`useMemo`を使うと良さそうです。

```diff ts: StudentSubmitForm.tsx
- const options = universityData.map(({universityCode, universityName}) => ({ universityCode, universityName}))
+ const options = useMemo(() => {
+  return universityData.map(({universityCode, universityName}) => ({ universityCode, universityName}))
+ }, [universityData])
```

APIとのやりとりの話とフォームの話は、混ぜない。
綺麗なコードを書こうとする上で、これは大事なことであるような気がします。

## 「setLoading、駆逐してやる!!この世から...一匹残らず...!」

ここからが本題です。
まず、さっきのコードをもう一度眺めてみます。

...。

改めて見ると、どうも`setLoading(boolean)`が三箇所も出てきて、くどいように感じます。今回は1つのAPIしか叩いてないのでまだ良い方です。3つのAPIを叩いたら、 `setLoading`を9回も目にすることになります。アプリケーション全体だと、その数は20や30では済まないかもしれません。

APIを叩くときって、ほぼ全てのケースで、叩く前に`setLoading(true)`、叩いた後に`setLoading(false)`すると思います。
それから、ちゃんとAPIからレスポンスが返ってきたら`data`を`state`にセットして、errorが起きたら`error`を`state`にセットしたりも毎回しているはず。

この辺のありがちな処理ってうまいこと共通化できないものでしょうか？

ちょっとやってみます。

```ts: useAsync.ts
const initialState = {data: undefined, loading: false, error: undefined}

export const useAsync = (asyncFn) => {
  const [state, setState] = useState(initialState);

  useEffect(() => {
    setState(prev => { ...prev, loading: true })

    asyncFn().then(
      (res) => {
        setState(prev => { ...prev, data: res, loading: false })

        return value;
      },
      (err) => {
        setState(prev => { ...prev, error: err, loading: false})
      }
    )
  }, [])

  return state;
}
```

こいつを、さっきの登録画面のページで使います。

```diff ts
import { useAsync} from "./useAsync"

const getUniversityList = async() => {
  const res = await fetch("/api/university/list", { method: "GET"})
  const universityList: University[] = await res.json()

  return universityList
}

const StudentSubmitPage: React.FC = () => {
+ const { data, loading, error } = useAsync(getUniversityList)

  return (
    <StudentForm universityData={data}>
  )
}
```

...!!
なんだこのスッキリ感は...!

やはり巨人は始祖ユミルだけで十分、`setLoading`は`useAsync`の中だけで十分なのです。

## 「なんのデータも!! 得られませんでした!!」

スッキリ書けたところで、しかし今度はリクエストが失敗したときのエラーメッセージを表示したくなってきました。
人類の最大の敵は、人類なのかもしれません...。

ちょっとやってみます。

```diff ts:StudentSubmitPage.tsx
import { useToasts } from "react-toast-notifications"

const StudentSubmitPage: React.FC = () => {
  const { data, loading, error } = useAsync(getUniversityList)
  const { addToast } = useToast()

+ useEffect(() => {
+   if(error !== undefined){
+     addToast("大学一覧の取得に失敗しました", {appearance: "error"} )
+   }
+ }, [error])

  return (
    <StudentForm universityData={data}>
  )
}
```

これで一見、問題なさそうですが、`useAsync`の中の`useEffect`を含めると、1回のAPIのやりとりで2回`useEffect`していることになります。
これは、あまりイケてないような気がします。

では先ほどの`useAsync`の中の`useEffect`の中に一緒に書いてしまうのはどうでしょうか？

```diff ts:useAsync.ts
(error) => {
  setState(prev => { ...prev, data: res, loading: false})
  // 役に立ったのですよね？息子の死は！！人類の反撃の糧になったのですよね！！！？
+ addToast("なんのデータも!! 得られませんでした!!", {appearance: "error"} )

  return error;
}
```

これだと、今度は、逆にエラーメッセージを出したくないときやエラーメッセージを変えたいときに柔軟に対応できません。

そこで、Errorが起きたときの処理を、外側からuseAsyncに渡してあげることで、この問題を解決してみたいと思います。

```diff ts:useAsync.ts
+ type Options = {
+  onError?: (error: Error) => void
+}

export const useAsync = (asyncFn, options?: Options) => {
    // ...(省略:さっきと同じ)...

  useEffect(() => {
    setState(prev => { ...prev, loading: true })

    asyncFn().then(
        // ...(省略:さっきと同じ)...
      (error) => {
      setState(prev => { ...prev, data: res, loading: false})
+      if(options?.onError){
+        onError(error)
+      }

        return error;
      }
    )
  }, [])

  return state;
}
```

```diff ts: StudentSubmitPage.tsx
const StudentSubmitPage: React.FC = () => {
  const { addToast } = useToast()
  const { data, loading, error } = useAsync(getUniversityList, {
    // 役に立ったのですよね？息子の死は！！人類の反撃の糧になったのですよね！！！？
+   onError: () => addToast("なんのデータも!! 得られませんでした!!", {appearance: "error"})
  })

  return (
    <StudentForm universityData={data}>
  )
}
```

これなら、`useEffect`は一回です。何より`Presentational`なコンポーネントに、ごちゃついたロジックが侵入していないのが気持ちいいです。

コードをシンプルにする話はここまでで、以降はUXの話題に移っていきます。

# APIとのやりとりをシンプルに保ちながら、ユーザー体験を向上させる方法

私事ですが、最近、ストレスを感じることがあります。勤怠管理用のアプリケーションが非常に使いづらいのです。何かを選択する度にデータを引っ張ってくるので、やたら画面にローディングが表示されて待たされまくります。

「いや、さっきのデータさっさと見せんかい！」
とハンジさんだってお怒りになるはずです。

とはいえ考えてみると、「いや、さっきのデータ見してくれたらええやん」的なアプリケーションはそこら中にある気がします。
例えば、、、

![slow](/images/slow.gif)

2ページ目から1ページ目に戻ったとき、すぐにさっきのデータを表示するのではなくて、もう1回「loading...」となってしまうWebサービス。
けっこうあります。

どうにかしてこのユーザー体験を変えられないでしょうか。

できるかもしれません、もし、APIから取得してきたデータをどこかに一時的に置いておくことさえできれば...。

データを......一時的に置いておく場所......?

......!!
JavaScriptには、そんな用途にぴったりな言語仕様がありました。

それは、`WeakMap`という弱い参照を持ったMap型のオブジェクトです。
以下、azuさんの[JS primer](https://jsprimer.net/basic/map-and-set/)から`WeakMap`に関する説明を抜粋します。

```javascript
// また、あるオブジェクトから計算した結果を一時的に保存する用途でもよく使われます。 次の例ではHTML要素の高さを計算した結果を保存して、2回目以降に同じ計算をしないようにしています。

const cache = new WeakMap();

function getHeight(element) {
  if (cache.has(element)) {
      return cache.get(element);
  }
  const height = element.getBoundingClientRect().height;
  // elementオブジェクトに対して高さをひもづけて保存している
  cache.set(element, height);
  return height;
}
```

この`WeakMap`をAPIを叩いたレスポンスの"結果を一時的に保存する用途"に使ったら、うまく問題が解決できそうです。
というわけで、先ほどの`useAsync`をベースに、`WeakMap`によるキャッシュの実装を加えた`useQuery`というカスタムフックを作ってみます。

```ts:useQuery.ts
import { useState, useEffect, DependencyList } from "react";

const cache = new WeakMap();

type CacheKey = { path: string; param: { page: number} };

const initialState = { data: undefined, loading: false, error: undefined };

export const useQuery = (
  cacheKey: CacheKey,
  asyncFn,
  deps
) => {
  const [state, setState] = useState(initialState);
  const [keys, setKeys] = useState([]);

  useEffect(() => {
    const _key = keys.find(
      (k) => JSON.stringify(k) === JSON.stringify(cacheKey)
    );
    const cachedData = cache.get(_key ?? {});

    if (cachedData) {
      return setState((prev) => ({ ...prev, data: cachedData }));
    }

    setState((prev) => ({ ...prev, loading: true }));
    asyncFn().then(
      (res) => {
        setState((prev) => ({
          ...prev,
          data: res,
          loading: false,
        }));
        cache.set(cacheKey ?? {}, res);
        setKeys([...keys, cacheKey]);

        return res;
      },
      (err) => {
        setState((prev) => ({ ...prev, error: err, loading: false }));
      }
    );
  }, [...deps]);

  return state;
};
```

これを次のような形で使います。

```diff ts:UniversityTablePage.tsx
const UniversityTablePage: React.FC = () => {
+ const { data, loading, error } = useQuery(
+   { path: "/api/universities?page=", param: { page } },
+   () => getUniversityList(page),
+   [page]
+ );
```

`useEffect`の依存配列に`page`を渡してあげているので、最初のレンダリングと`page`の値が変わったタイミングでAPIがコールされます。

この例では`useAsync`の`key`の引数に、`{ path:"/api/university/list", param: { page: ページ数}}`を渡してあげているので、`WeakMap`の挙動は次のようになります。

```typescript
// 1ページ目読み込み中は、WeakMapの中身は空っぽ
const cache = new WeakMap();
↓
// 1ページ目の読み込みが終わると、WeakMapには1ページ目のデータが入る
const cache = new WeakMap([[{ endpoint: 1, param: 1 },  1ページ目のデータ]]);
↓
// 2ページ目の読み込みが終わると、WeakMapには2ページ目のデータが加わる
const cache = new WeakMap([
  [{ path: "/api/university/list", param: { page: 1 }}, 1ページ目のデータ],
  [{ path: "/api/university/list", param: { page: 2 }}, 2ページ目のデータ]
]);
```

この状態で2ページ目から1ページ目に戻ると、`{ path: "/api/university/list", param: { page: 1 }}`というキーに1ページ目のデータが格納されているので、`cache.get(key)`の返す値は、1ページ目のデータになります。
取得し直すまでもなく、データは、もう、既にそこにあるので、瞬時に表示されます。
改善された動画がこちらです↓

![fast](/images/fast.gif)

これで、ユーザーをイラつかせるような無駄なローディングの群れに別れを告げられました。それはまるで、

> 自由をもたらす、鐘の音のようだったよ...。

しんみり。

ここまでのサンプルコードはGitHubにあげて、Web上にデプロイしました。実際に触ってみて、挙動の違いを確かめてみてください。また上記の例では、説明の便宜上`typescript`の型をサボりましたが、GitHubにはちゃんと型をつけたものをアップしてあります。

[進撃のデータフェッチサンプルWeb](https://attack-on-react-query.vercel.app/)
[進撃のデータフェッチサンプルGitHub](https://github.com/t-keshi/attack-on-react-query)

# そして、データフェッチングライブラリへ

APIのやり取りはシンプルになり、不要なローディングに別れを告げて、
ようやく人類は自由と平和を取り戻した...

...か、に見えましたが、アニメというものは1クールが終わっても、また次のシーズンで新たな物語が展開されていくものらしいです。
進撃の巨人だって４期もありますからね。

そして、ここまで書いてきたコードなのですが、実は目を瞑ってきた様々な問題と、ツッコミどころが満載です。

例えば、キャンセル処理です。

あるページから別のページへ遷移すると、新しいデータ取得が走ります。しかし、このときブラウザは、前ページのデータ取得の処理を途中で止めることができるわけではありません。ちゃんとやろうとするなら、[AbortController](https://developer.mozilla.org/ja/docs/Web/API/AbortController)を使ってリクエストのキャンセル処理を書いてあげるべきです。

また、何かしらのデータが更新された後など、逆に古いデータが表示されると困るケースが出てきます。「そのキャッシュ、古くなったんで、使わないでください」といった操作ができるFunctionが必要です。このFunctionを仮に、`invalidateQueries`と呼んでみます。

では、粘り強く頑張って、キャンセル処理と`invalidateQueries`を書いていきましょうか。
でも実際それってすごくハードだし、自分で書いたコードなど所詮は汚いマーレ人です。

どこかに、良いライブラリが転がっていれば、それが一番安全で、かつ、手っ取り早いのですが...。

どこかに...、良いライブラリが..................。

......!!

[ReactQuery](https://tanstack.com/query/v4)

[SWR](https://swr.vercel.app/ja)

ぬぬ！
こいつらですか、もしかして。

# まとめ

というわけで、色々と自分で頑張ったあげく、最終的にライブラリに出会う話でした。データフェッチングライブラリを手書きするハンズオンみたいになってしまいましたが、車輪の再発明なので、普通に有名どころのライブラリを使っていくのが好手だと思います。

すでに紹介したもの以外にも、有名どころのライブラリには、次のようなものがあります。
REST(or GraphQL): `ReactQuery`, `SWR`[^1]
GraphQL: `Relay`, `Apollo`, `URQL`
こうしたライブラリを使う方がスッキリ書けるし、読み込みの体験もスマートにできます。

余談になりますが、仮にライブラリを導入しないことにしたとしても、その背後にあるアイデアだけでも盗んでおくのはどうでしょうか。

特に、ReactQueryの作者"Tanner Linsley"が示した`ClientState`と`ServerState`を分ける思想は、非常に参考になります。[React Query: It’s Time to Break up with your "Global State”!](https://www.youtube.com/watch?v=seU46c6Jz7E)で詳しく解説されています。こうしたStateの分離戦略は以下の記事でも紹介されており、昨今、市民権を得つつある気配がします。
[「3種類」で管理するReactのState戦略](https://zenn.dev/yoshiko/articles/607ec0c9b0408d)
[スコープとライフタイムで考えるReact State再考](https://zenn.dev/akfm/articles/react-state-scope)

長くなってすみません、ようやく終わりです！

最後に宣伝です。
先日、エンジニアのための書評サービス「積読ミシュラン」をリリースしました。`Next.js`✖️`SWR`✖️`GraphQL`な技術スタックになっています。興味があれば覗いていってください。
[積読ミシュランWebサイト](https://tsundoku-michelin.vercel.app/books-list/1)
[積読ミシュランGitHub](https://github.com/t-keshi/tsundoku-michelin)

最後までお読みいただきありがとうございました！

[^1]: 今回は`ReactQuery`と`SWR`を一括りにして紹介してしまいましたが、中身は大きく異なるライブラリです。npmを見たら、UnpackedSizeは`ReactQuery`は1.81MBなのに対し、`SWR`は231kBでした。`ReactQuery`の方が、コードの量も数倍多いです。ただ、それだけ`ReactQuery`の方が親切に色々やってくれるし、充実した機能を持ってるということでもあります。親切なライブラリを取るか、シンプルなライブラリを取るか。これが技術選定の決め手になると思います。
