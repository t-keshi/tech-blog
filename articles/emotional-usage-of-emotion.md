---
title: '君のemotionを解き放て！－CSSinJSの歩き方－'
emoji: '👩‍🎤'
type: 'tech'
topics: [react, emotion, CSSinJS, styledcomponents]
published: true
---

![emotion](/images/emotion.png)

↑この子の名前は"emotion"。

`styled-component`と並んで、最も有名なCSS-in-JSのライブラリの１つです。

`CSS-in-JS`とは、JavaScriptを使ってスタイルを書く方法です。`CSS-in-JS`を使うことにより、CSSにありがちな名前の衝突やスタイルの複雑な優先順位の問題に別れを告げることができます。そして、より宣言的に、より再利用可能な方法でスタイルを記述できます。

今日はそんなemotionだけにエモい話です。`emotion`に限らず、`styled-components`でも`jss`でも、CSS-in-JS全般に興味のある方に読んでいただけると大変嬉しいです。

# あるフロントエンドの悩み

`CSS-in-JS`はまだまだ歴史が浅く、ベストプラクティスが確立されていません。そのため、同じ`CSS-in-JS`であっても人によって使い方は様々で、悪い意味で書く人の個性が出てしまいます。ある書き方はメンテナンスしやすくしますが、また、ある書き方は`emotion`のメリットを十分に引き出せません。

どうせなら、emotionを解き放ちたい...!

今回は、Mui(MaterialUI)とChakraUIからヒントをもらって、`emotion`を解き放つ方法を考えてみました。

これがCSS-in-JSのベストプラクティスだ！
なんていうつもりはありませんが、何らかの参考になれば幸いです。

# 1. インラインで書くことを恐れない

最初に、一般的にCSSでは、長らく「インラインで書くことは悪」とされてきましたが、こと`CSS-in-JS`においてはむしろ、「インラインで書くことは正義」なのではないかと主張します。

以下の例は、[Mui](https://mui.com/system/basics/)から持ってきました。

![statComponent](/images/statComponent.png)


```javascript
const StatWrapper = styled('div')(
  ({ theme }) => `
  background-color: ${theme.palette.background.paper};
  box-shadow: ${theme.shadows[1]};
  border-radius: ${theme.shape.borderRadius}px;
  padding: ${theme.spacing(2)};
  min-width: 300px;
`,
);

const StatHeader = styled('div')(
  ({ theme }) => `
  color: ${theme.palette.text.secondary};
`,
);

const StyledTrend = styled(TrendingUpIcon)(
  ({ theme }) => `
  color: ${theme.palette.success.dark};
  font-size: 16px;
  vertical-alignment: sub;
`,
);

const StatValue = styled('div')(
  ({ theme }) => `
  color: ${theme.palette.text.primary};
  font-size: 34px;
  font-weight: ${theme.typography.fontWeightMedium};
`,
);

const StatDiff = styled('div')(
  ({ theme }) => `
  color: ${theme.palette.success.dark};
  display: inline;
  font-weight: ${theme.typography.fontWeightMedium};
  margin-left: ${theme.spacing(0.5)};
  margin-right: ${theme.spacing(0.5)};
`,
);

const StatPrevious = styled('div')(
  ({ theme }) => `
  color: ${theme.palette.text.secondary};
  display: inline;
  font-size: 12px;
`,
);

return (
  <StatWrapper>
    <StatHeader>Sessions</StatHeader>
    <StatValue>98.3 K</StatValue>
    <StyledTrend />
    <StatDiff>18.77%</StatDiff>
    <StatPrevious>vs last week</StatPrevious>
  </StatWrapper>
);
```

これは「インラインで書くことは悪」の立場に基づく一般的な`CSS-in-JS`の使い方です。しかし、本当に、読みやすいでしょうか。もう一度よく眺めてみてください。

上記のコードを読みながら、これはどんなコードなんだろうと理解するために、視線は上下に何度も移動しませんでしたか。

目は最初に下まで行って`return`の後に`<StatWrapper>`と書いてあるのを見ると、一番上の`const StatWrapper = styled('div')...`と書いてあるところまで戻ってきます。それからまた下へ行き`<StatHeader>`と書いてあるのを見つけると、上の`const StatHeader = styled('div')...`と書いてあるところに戻って...、と上限運動を繰り返します。

つまり、この書き方だと、上へ下へと行ったり来たりしながら読む必要があり、あまり直感的な書き方ではないと言えそうです。

加えて、コンポーネントの名前をつけるのにも難儀するはずです。上記デザインサンプルの`Sessions`の文字にあたる箇所は、`StatHeader`と呼ぼうか、それとも、`StatTitle`がいいだろうか。。そのようなことをいちいち考えるのは、面倒です。

では、次の例はどうでしょうか。

```javascript
<Box
  bgColor="'background.paper"
  boxShadow={1}
  borderRadius={1}
  p={2}
  minWidth={300}
>
  <Box color="text.secondary">Sessions</Box>
  <Box color="text.primary" fontSize={34} fontWeight="medium">
    98.3 K
  </Box>
  <Box
    component={TrendingUpIcon}
    color="success.dark"
    fontSize={16}
    verticalAlign="sub"
  />
  <Box
    color="success.dark"
    display="inline"
    fontWeight="medium"
    mx={0.5}
  >
    18.77%
  </Box>
  <Box color="text.secondary" display="inline" fontSize={12}>
    vs. last week
  </Box>
</Box>
```

今度は、上から下までざっと一回なめるだけで、どんなふうにスタイルを当てているか一発で理解できたのではないでしょうか。先ほどの例より、すんなり把握できたはずです。命名に手こずる必要もありません。

ちなみにEmotionをこのような使い方で書くには、

``` javascript
const Box = styled.div(props => ({
  display: 'flex',
  boxShadow: props.boxShadow,
  borderRadius: props.borderRadius
  padding: props.padding
  minWidth: props.minWidth
  ...
}))
```

のように書いていけばいいのですが、`mixin`を使うともっと簡潔に書けます。この点については、[styled-components と csstype で型安全な Chakra UI っぽいコンポーネントを作る](https://zenn.dev/suzukalight/articles/styled-components-csstype)という記事が非常に参考になります。また、`mixin`を自分で書くのが面倒であれば、[styled-system](https://styled-system.com/)を使うという手もあります。


## 1.1 ところで、構造とスタイルの分離はどうしたの？

そうですよね。いやぁ、わかります。

私も、長らく、こういうインラインで書く方が便利だと思いながらも、

> (でも、本当は構造とスタイルを分離させて書いた方がいいんだよね...)

と義務感を感じていました。多少読みづらくても、教科書的にはスタイルを分けた方が正しいのだろう、と。

しかしこの頃、「構造とスタイルの分離」という概念からそろそろ旅立つときが来たのではないか、と考え始めました。

理由は、Reactです。

React以前は「関心の分離」という概念のフロントエンドにおける実践は、イコール、HTMLの構造とCSSのスタイルを分ける「構造とスタイルの分離」でした。

そこに、Reactは「コンポーネント」という単位で分ける新しい分離方法を持ち込みました。そもそも、JavascriptでHTMLを生成する発想自体が、ファイル形式ごとに分割する従来の方法とは一線を画しています。ファイル形式ごとに分けていたのは、そういうふうにしか分けられなかったからであって、別の手段を手にした今となってはそれを忠実に守る必要はないのかもしれません。

今やファイル形式ではなく、コンポーネントごとにコードの関心の分離を整理した方が、ずっと管理しやすいものになります。


> 分離すべきは「あるコンポーネントの関心事と他のコンポーネントの関心事」であって、「あるコンポーネント内のHTMLとスタイル」ではありません。

と[Kabukuの記事](https://www.kabuku.co.jp/developers/tailwindcss-ecosystem)にあるように、「コンポーネントの分離」は「ファイル形式の分離」に勝ります。故に、もはや「構造とスタイルの分離」に固執する必要はないのではないか、というのが私の意見です。

...ということで、ガンガン、インラインで書いていきましょう！

参考：

[フロントエンドにおける「関心の分離」は間違っていた](https://scrapbox.io/fsubal/%E3%83%95%E3%83%AD%E3%83%B3%E3%83%88%E3%82%A8%E3%83%B3%E3%83%89%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8B%E3%80%8C%E9%96%A2%E5%BF%83%E3%81%AE%E5%88%86%E9%9B%A2%E3%80%8D%E3%81%AF%E9%96%93%E9%81%95%E3%81%A3%E3%81%A6%E3%81%84%E3%81%9F)
[Rethinking Separation of Concerns with React](https://www.dottedsquirrel.com/separation-of-conern-react/)

## 2. 多彩なコンポーネントを揃える

先ほどのインラインで書く場合の例ですが、

```javascript
<Box
  bgColor="'background.paper"
  boxShadow={1}
  borderRadius={1}
  p={2}
  minWidth={300}
>
  <Box color="text.secondary">Sessions</Box>
  <Box color="text.primary" fontSize={34} fontWeight="medium">
  ...(省略)...
```

と、div地獄ならぬBox地獄ではないですが、何だかBoxBoxしていて、まだ読みづらさがある気がします。
次のように書けたら、より読みやすくはないでしょうか。

```javascript
<Card minWidth={300}>
  <Typography color="text.secondary" fontSize="md">Sessions</Typography>
  <Typography color="text.primary" fontSize="lg" fontWeight="md">
    98.3 K
  </Typography>
  <TrendingUpIcon
    color="success.dark"
    fontSize="sm"
    verticalAlign="sub"
  />
  <Typography
    color="success.dark"
    fontWeight="md"
    inline
    mx={0.5}
  >
    18.77%
  </Typography>
  <Typography color="text.secondary" fontSize="xs">
    vs. last week
  </Typography>
</Card>
```

`Box`という単一のコンポーネントを使う代わりに、`Card`や`Typography`という多彩なコンポーネントを使うことで、ぐっと構造が捉えやすくなったのではないかと思います。

ここで使っている`Card`のようなコンポーネントは、例えば次のように、予め揃えておくと楽です。

``` javascript
const Card = styled.div(props => ({
  display: 'flex',
  boxShadow: props.boxShadow || 1,
  borderRadius: props.borderRadius || 1,
  padding: props.padding || 2,
  minWidth: props.minWidth || 100

  &hover: {
    bosShadow: props.boxShadow + 1 || 2,
  }
}))
```

予めデフォルトのスタイルを書いておいて、上書きしたいところだけ、uiを使う側のコンポーネントで書くようにすれば、ずっとスッキリ書くことができます。

インラインで書くのが良いといっても、極端な話ですが、

```javascript
<Card
  display='flex'
  boxShadow={1}
  borderRadius={1}
  padding={2}
  minWidth={200}
  maxWidth={400}
  height={200}
  borderColor="grey"
  // {...その他、20行くらいのprops...}
>
```

となってしまっては、非常に見にくいです。これをやるくらいなら、インラインではなくcssを分離した方がまだ良さそうです。言い換えると、インラインで書くには、インラインで書きやすいようにデフォルトのスタイルを十分に持った多彩なコンポーネントを用意しておく必要があります。

[ChakraUI](https://chakra-ui.com/)には、Layoutのためのコンポーネントだけで、`AspectRatio`, `Box`, `Center`, `Container`, `Flex`, `Grid`, `SimpleGrid`, `Stack`, `Wrap`と９つもあります。これくらい品揃えが豊富だと、ほとんどスタイルを書かなくても画面が組めるので、スピーディーに開発することができます。

[AtomicDesign](https://atomicdesign.bradfrost.com/)でいうと、いきなり`pages`や`template`から書き始める前に、`atoms`や`molecules`層に十分なコンポーネント群を準備しておくことで後々の開発効率が上がりそうです。[コンポーネント駆動開発](https://qiita.com/UCLab1421/items/1c4e4acfdc785dbfa269)的な感じで。

## 3. 解像度を荒くする

ところで、先ほどの例では`fontSize={34}`とか`fontSize={16}`になっていたところを、こっそり`fontSize="lg"`とか`fontSize="md"`に変えていました。これは、「バラつきがひどい問題」を解消するための、ちょっとした工夫です。


たぶん、誰もが一度は、「バラつきがひどい問題」に遭遇したことがあると思います。

1. デザインのバラつき

同じように「小さい文字で」と指定されても、人によっては`fontSize="12px"`と書きますし、人によっては`fontSize="10px"`と書くかもしれません。`fontSize`のバリエーションがありすぎると、cssを管理しにくくなります。

2. サイズ指定方法のバラつき

サイズを`12px`,`9pt`,`0.75em`,`0.75rem`,`75%`など、人によって様々な方法で指定するかもしれません。[fontSizeはremを使うのがいい](https://qiita.com/butchi_y/items/453654828d9d6c9f94b0)と考えて、チーム全員で`rem`を使うようにルールを決めたとしても、メンバー全員が忠実にそのルールを守るとは限りません。きっと、うっかり`px`を使う人が跡を絶たないはずです。

意図的に解像度を荒くすれば、上記２つのバラつき問題は解消できます。もし、fontSizeが`xs`,`sm`,`md`,`lg`,`xl`の5つしか指定できないのであれば、5つのバリエーションしか生まれようがありません。それは、次のように実装することで実現可能です。

```typescript
type TypographyProps = {
  fontSize: 'xs' | 'sm' | 'md' | 'lg' | 'xl';
  inline?: boolean;
  textAlign?: React.CSSProperties['textAlign'];
} & { theme: Theme };

const fontSizes = {
  xs: "0.75rem",
  sm: "0.875rem",
  md: "1rem",
  lg: "1.5rem",
  xl: "2.625rem",
};

const Typography = styled.p((props: TypographyProps) => ({
  fontSize: fontSizes[props.fontSize],
  display: props.inline ? 'inline' : 'block',
}));
```

これで`fontSize`のバラつきは解消されましたが、しかし、まだ他にもバラつきが残ります。というのも例えば、`fontSize`が5種類、`fontWight`が5種類合ったとしたら5✖️5=25種類の文字が生まれることになります。そこに`letterSpacing`や`lineHeight`も加わったら、バラつきはもっと大きくなります。

さらにバラつきを減らしたいときは、`emotion`の`theming`の出番です。[Muiのvariantのアイデア](https://mui.com/material-ui/api/typography/)を参考にして、次のように実装することができます。以下では、`Typography`を一例に取りますが、`Button`や`Input`などのコンポーネントにも`variant`を作るのがおすすめです。


まずは、themeを作って、projectのルートに近い層でラッピングします。

```typescript
import { ThemeProvider } from '@emotion/react';

const theme = {
  typography: {
    h1: {
      fontSize: '3rem',
      lineHeight: 1.333,
      fontWeight: 500,
      letterSpacing: '-0.02083em',
    },
    h2: {
      fontSize: '2.125rem',
      lineHeight: 1.353,
      fontWeight: 500,
      letterSpacing: '0em',
    },
    h3: {
      fontSize: '1.5rem',
      lineHeight: 1.333,
      fontWeight: 500,
      letterSpacing: '0.02083em',
    },
  },
  palette: {
    text: {
      primary: 'rgba(0, 0, 0, 0.87)',
      secondary: 'rgba(0, 0, 0, 0.6)',
    },
  },
};

const App: React.FC = () => (
  <ThemeProvider theme={theme}>
    // ここにコンポーネント
  </ThemeProvider>
);
```

typescriptの場合は、別途、[themeをdeclare](https://emotion.sh/docs/typescript#define-a-theme)する必要があります。

そして、このthemeを使って次のように書くことができます。`variant`の次いでに、`color`もthemeを使って指定できるようにします。

``` typescript
type TypographyProps = {
  variant: 'h1' | 'h2' | 'h3';
  color: 'text.primary' | 'text.secondary';
} & { theme: Theme };


const generateFontColor = (theme: Theme) => ({
  'text.primary': theme.palette.text.primary,
  'text.secondary': theme.palette.text.secondary,
});

const Typography = styled.p((props: TypographyProps) => {
  const { color, variant, theme } = props;

  return {
    ...theme.typography[variant],
    color: generateFontColor(theme)[color],
  };
});
```

これで、綺麗にバラつきは無くなりました。エンジニアにもデザイナーにも優しい設計ではないでしょうか。

とはいえ、解像度をどこまで荒くするかは、プロジェクト次第です。場合によっては`fontSize="md"`と書けるようにした方が便利かもしれませんし、別のケースでは`variant="h1"`と指定するようにしたほうが早いかもしれません。

この辺りをプロジェクトに合わせて、柔軟に設定できるのが`emotion`の旨味の１つと言えそうです。

おっ、いい感じにまとまりましたね！


# おわりに

かなり独断と偏見でつらつらと書いてしまいましたが、突っ込みどころがあれば教えてください。特に構造とスタイルの分離は、他の方の意見をお伺いしてみたいと思っているところです。お気軽にコメントお待ちしております🙇‍♂️

最後に、趣味で`emotion`をベースにしたUIライブラリを開発中です。よろしければ、ちらっと覗いてやってください。

[github](https://github.com/t-keshi/cheek-ui)
[storybook](https://6270b164bcbdf4004a31f1a3-ccdinutnnd.chromatic.com/?path=/story/inputs-button--all)