# 参照
https://nextjs.org/learn

## 対象
- スタイリング
- 最適化
- ルーティング
- データフェッチ
- 検索、ページ区切り
- データの変更
- エラー処理
- フォーム検証、アクセシビリティ
- 認証
- メタデータ

# デモ
`nextjs-dashboard` をいじりながら進める。

## フォルダ構造 (一例)
- /app
  - /lib: util, fetcher など
  - /ui: UI コンポーネント
  - /public: 静的アセット
  
# スタイリング
## tailwind
tailwindcss util class を使用する
```css
/* global.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

## css modules
異なるスタイルファイル間での衝突を防ぐことが期待できる
```css
/* home.module.css */
.shape {
  height: 0;
  width: 0;
  border-bottom: 30px solid black;
  border-left: 20px solid transparent;
  border-right: 20px solid transparent;
}
```

## clsx
クラス名を条件分岐で切り替えできる

# フォント
フォントはパフォーマンスに強く影響する
- **累積レイアウトシフト**：Google が SEO の指標に用いる、フォールバックフォントからシステムフォントに置き換えた際に生じるレイアウトのズレの大きさ

Next は、`next/font` モジュールによってアプリケーション内のフォントを自動的に最適化する

```ts
// fonts.ts
import { Inter } from 'next/font/google';

// subsets: 使用する文字だけを絞り込んで用いる
export const inter = Inter({ subsets: ['latin'] });
```

```ts
// layout.tsx
import '@/app/ui/global.css';
import { inter } from '@/app/ui/fonts';
 
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={`${inter.className} antialiased`}>{children}</body>
    </html>
  );
}
```

# 画像
`/public` に設置
`next/image` の `<Image>` を使用することで、レイアウト最適や各種最適化がつく

```jsx
<Image
  src="/hero-desktop.png"
  width={1000}
  height={760}
  className="hidden md:block"
  alt="Screenshots of the dashboard project showing desktop version"
/>
```

## レスポンシブ
プレフィクスを設けることで、条件に応じた場合のみ適応するクラスが設定できる
このとき、**プレフィクスがない側がモバイルに適用されるクラスである**点に注意する
```jsx
// for Desktop
<Image
  src="/hero-desktop.png"
  width={1000}
  height={760}
  className="hidden md:block"
  alt="Screenshots of the dashboard project showing desktop version"
/>

// for Mobile
<Image
  src="/hero-mobile.png"
  width={560}
  height={620}
  className="block md:hidden"
  alt="Screenshots of the dashboard project showing mobile version"
/>
```

# ルーティング
Next.js では **ファイルシステムルーティング** を採用している

```
- app (/)
  - dashboard (/dashboard)
    - invoices (/dashboard/invoices)
```

## page.tsx
各フォルダ内に設置された `page.tsx` が、各ルートのページコンポーネントとして扱われる

逆に言えば、app 内にフォルダが作成されても、 `page.tsx` （または`route.ts`）を設置しなければルーティングに反映されることはない
https://nextjs.org/docs/app/getting-started/project-structure#colocation

また、`_` を頭につける、`()` で囲むことで明示的にルーティングから除外することもできる（例：`_hoge`, `(hoge)`）

## layout.tsx
各フォルダ内に設置された `layout.tsx` は、そのフォルダおよびフォルダ内のすべてのネストされたフォルダ内の `page.tsx` の規定レイアウトとなる

レイアウトは **ナビゲーション時に再レンダリングされない**
つまり、ページ遷移前後で同一のパーツをサーバから取り寄せることがなく、不要な処理・通信が発生しない

# Vercel と DB
Vercel には DB 連携機能ができたっぽい（ダッシュボードから supabase などの外部リソースを捜査できた）
https://supabase.com/partners/integrations/vercel

## リクエストウォーターフォール
リクエストを前から順に処理していく一連の流れ
同期的に処理していくため、重い処理が挟まると後続の処理も遅れる

```ts
const hoge = await fetchHoge();
const huga = await fetchHuga();  // fetchHoge の終了を待つ
const piyo = await fetchPiyo();  // fetchHuga の終了を待つ
```

## 並列データフェッチ
JavaScript では `Promice.all` で実現できる

```ts
const data = await Promise.all([
  fetchHoge,
  fetchHuga,
  fetchPiyo
]);
```

# レンダリング
## 静的レンダリング
データの取得・レンダリングを **ビルド時・データ再検証時** にサーバ上で行うこと
高速で軽量、SEO に強いが、表示データが変わりやすいアプリには向かない

## 動的レンダリング
データの取得・レンダリングを **毎リクエスト時** にサーバ上で行うこと
リアルタイムなデータを反映するのに向いているが、SEO 等に弱い

動的レンダリングでページをレンダリングした際、ページが最初に表示されるのは **最後のデータフェッチが終了した後** になる

# ストリーミング
データを分割して送信する技術
動的レンダリングの表示が遅れる問題を解決するのに使える

Next.js ではルートフォルダ内に設置された `loading.tsx` がローディング中に用いられる

```tsx
// app/hoge/loading.tsx

// /hoge にアクセスした直後、データフェッチが終了するまで表示される
export default function Loading() {
  return <div>Loading...</div>;
}
```

## Suspense
React から提供される、部分的ストリーミングのためのコンポーネント
`fallback` パラメータにフォールバックコンポーネントを渡すことで、ローディング表示が実装できる

```tsx
<Suspense fallback={<RevenueChartSkeleton />}>
  <RevenueChart />
</Suspense>
```

# クエリパラメータ
Next.js では、CSR/SSR 別にクエリパラメータへアクセスする方法が用意されている

## CSR
`useSearchParams` を使用する

```ts
const searchParams = useSearchParams();
```

## SSR
ページコンポーネントの props は `searchParams` という値を受け取れるので、これを使用する

```tsx
export default async function Page(props: {
  searchParams?: Promise<{
    query?: string;
    page?: string;
  }>;
}) {}
```

# デバウンス
アクティブに更新するようなアプリケーションに検索クエリを埋め込む場合（例：入力ごとに検索結果を更新する検索バー）、クエリが高頻度で送信されることを避ける必要がある
この手法を **デバウンス** とよぶ

1. trigger: デバウンスが発生すると、タイマーが開始される
2. wait:    タイマー期限前にイベントが発生した場合リセット
3. execute: タイマーが終了すると、デバウンスされた関数が実行

たとえば検索バーの例では、感覚を300ミリ秒に設定した場合、次の入力が来ないか300ミリ秒待ってから検索を実行する　といった具合になる


Next.js では `use-debounce` という外部ライブラリを推奨している
https://www.npmjs.com/package/use-debounce

# React Server Action
React に提供される、 API を用意せずともサーバ上で直接非同期コードを実行できる手法
これにより、**Progressive Enhancement（斬新的拡張）**：html→css→js の順に読み込み、読み込まれたものから適用するフローにおいて、js が読み込まれていない状況（例：ネット回線が遅い）でもアプリの一部が正常に動作する、などのメリットがある

```tsx
// Server Component
export default function Page() {
  // Action
  async function create(formData: FormData) {
    'use server';
 
    // Logic to mutate data...
  }
 
  // Invoke the action using the "action" attribute
  return <form action={create}>...</form>;
}
```

## Next.js のキャッシュと Server Components
Next.js では動作高速化のためにキャッシュを使用するが、form action による画面遷移が発生した場合、更新したカラム情報がキャッシュに反映されず、実際のデータと異なるデータ状態を表示してしまうことがある
一度キャッシュされてしまうと、キャッシュを消去する操作を行うまで同じ画面が表示され続けるため、知らないままでは非常に厄介な機能となる

これを防ぐため、データ更新後は必ず `revalidatePath` `revalidateTag` などを使用して再検証を行う

## revalidatePath
特定のルートページにおけるキャッシュを再検証する

```ts
// ユーザー一覧ページのキャッシュを再検証
revalidatePath("/users");
```

## revalidateTag
Next.js の fetch には、オプションとして **タグ** というものが付与できる
これを付与しておくことで、`revalidateTag` による一括再検証が可能になる

```ts
// いくつかの fetch に対して "user" タグを付与する
await fetch("http://localhost:3000/users", {
  next: { tags: ["user"] },
});
await fetch("http://localhost:3000/users/hogehoge", {
  next: { tags: ["user"] },
});

// "user" タグの fetch 結果キャッシュに対し、再検証を行う
revalidateTag("user");
```

# エラーハンドリング
Next.js では、ルートごとに `error.tsx` を設置することでフォールバックUIを定義できる

```tsx
export default function Error({
  error,   // Error インスタンス
  reset,   // エラーをリセットする関数
}: {}
```

## notFound() と not-found.tsx
404 エラーを独自に処理する場合、`notFound()` と `not-found.tsx` が役に立つ
`not-found.tsx` は `error.tsx` と同様、ルートごとに設置できる

```tsx
import { notFound } from 'next/navigation';

// ...

if (!invoice) {
  notFound();   // -> not-found.tsx を探して表示する
}
```


# アクセシビリティ
eslint の `eslint-plugin-jsx-a11y` は、コードベース内のアクセシビリティに関する問題（例：`alt` のない画像、`aria-*`, `role` の誤用）を早期に発見することができる
Next.js の `Image` などにも適用される


# フォームの検証
たとえば空のデータをフォームとして送信したとき、なんの対応もしていないとエラーページが表示される
これを防ぐため、フォームの検証を追加することができる

## クライアント側
`required` を使用する

```tsx
<input
  id="amount"
  name="amount"
  type="number"
  required   // add
/>
```

## サーバ側
クライアント側での対策は「行儀のよい」ユーザーの誤送信を防ぐのみで、API へ直に送信するようなリクエストには対応できない
これを防ぐため、サーバ側検証を行う

React/Next では `useActionState` を使用する

```ts
// action.ts


// フォーム状態を示す型を定義（ここでは Zod が返すエラー構造をトレースした型）
type State = {
  errors?: {
    customerId?: string[];
    amount?: string[];
    status?: string[];
  };
  message?: string | null;
}

// [in action function]
// Zod による parse に safeParse を用い、エラーを帰り値として返すようにする
const validatedFields = CreateInvoice.safeParse({
  customerId: formData.get('customerId'),
  amount: formData.get('amount'),
  status: formData.get('status'),
})

if (!validatedFields.success) {
  // 帰り値の型は State と同一
  return {
    errors: validatedFields.error.flatten().fieldErrors,
    message: "Missing Fields. Failed to Create Invoice."
  }
}
```

```ts
// page.tsx


// 初期状態を定義
const initialState: State = { message: null, errors: {} };

// useActionState に任意の関数と初期状態を渡す
const [state, formAction] = useActionState(createInvoice, initialState);

// これを state に渡す
<form action={formAction} aria-describedby="form-error">

// あとは任意の場所で任意のエラーを表示すればoK
<div id="customer-error" aria-live="polite" aria-atomic="true">
  {state.errors?.customerId &&
    state.errors.customerId.map((error: string) => (
      <p className="mt-2 text-sm text-red-500" key={error}>
        {error}
      </p>
    ))}
</div>
```

