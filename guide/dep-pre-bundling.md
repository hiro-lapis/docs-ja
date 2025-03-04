# 依存関係の事前バンドル

初めて、`vite` を実行すると、次のメッセージが表示される場合があります:

```
Pre-bundling dependencies:
  react
  react-dom
(this will be run only when your dependencies or config have changed)
```

## その理由は？

これは、Vite が「依存関係の事前バンドル」を実行しています。

このプロセスには 2 つの目的があります:

1. **CommonJS と UMD の互換性:** 開発中の Vite のコードは ECMAScript モジュールとして提供しています。そのため、Vite は、CommonJS または、UMD を ESM に変換する必要があります。CommonJS の依存関係を変換する場合、Vite はインポート文をスマート分析を実行してエクスポートが動的に割り当てられていても、CommonJS モジュールは期待通りに動作します。(例 React):

```js
   // works as expected
   import React, { useState } from 'react'
```

2. **パフォーマンス:** Vite は、多くの内部モジュールを持つ ESM の依存関係を単一のモジュールに変換して、その後のページロードのパフォーマンスを向上させます。いくつかのパッケージでは、ECMAScript モジュールのビルドを、相互にインポートする別々のファイルとして出力します。

一例として [`lodash-es`](https://unpkg.com/browse/lodash-es/) には、600 以上の内部モジュールがあります。`import { debounce } from 'lodash-es'` をすると、ブラウザは 600 以上の HTTP リクエストを同時に処理します！　サーバ側では問題なく処理していても、大量のリクエストによりブラウザ側でネットワークの混雑が発生し、ページの読み込みが著しく遅くなってしまいます。

事前に `lodash-es` を単一のモジュールにバンドルすることにより HTTP リクエストは 1 つだけで済むようになりました。

これは開発モードでのみ適用されることに注意してください。

## 自動依存関係の検出

既存のキャッシュが見つからない場合、Vite はソースコードをクロールし、依存関係のインポートを自動的に検出します。（すなわち、`node_modules` から解決されることを期待されている "bare imports"）を探し、事前バンドルのエントリポイントとして使用します。事前バンドルは `esbuild` で実行されるので、通常は非常に高速です。

サーバを起動したあと、キャッシュにない新しい依存関係のインポートに遭遇した場合は、Vite は、依存関係管理ツールによる再バンドリングプロセスを実行し、ページをリロードします。

## モノレポとリンクされた依存関係

モノレポの設定では、依存関係は同じリポジトリからのリンクされたパッケージの可能性があります。Vite は `node_modules` から解決されない依存関係を自動的に検出し、リンクされた依存関係をソースコードとして扱います。リンクされた依存関係をバンドルしようとはせず、代わりにリンクされた依存関係のリストを分析します。

ただしこの場合、リンクされた依存関係が ESM としてエクスポートされている必要があります。そうでない場合は、[`optimizeDeps.include`](/config/#optimizedeps-include) と [`build.commonjsOptions.include`](/config/#build-commonjsoptions) に依存関係を追加して、設定することができます。

```js
export default defineConfig({
  optimizeDeps: {
    include: ['linked-dep']
  },
  build: {
    commonjsOptions: {
      include: [/linked-dep/, /node_modules/]
    }
  }
})
```

リンクされた依存関係を変更する場合は、`--force` コマンドラインオプションを指定して開発サーバーを再起動すると、変更が有効になります。

::: warning Deduping
リンクされた依存関係は、依存関係の解決方法の違いにより、推移的な依存関係が不正に重複排除されることがあり、実行時に問題が発生することがあります。この問題につまずいた場合は、リンクされた依存関係に対して `npm pack` を使って修正します。
:::

## 挙動のカスタマイズ

デフォルトの依存関係発見の経験則は、必ずしも望ましいとは限りません。リストから依存関係を明示的に含めたり除外したりする場合は、[`optimizeDeps` 設定オプション](/config/#依存関係の最適化オプション)を使用してください。

`optimizeDeps.include` または `optimizeDeps.exclude` の一般的な使用例は、ソースコードで直接検出できないインポートがある場合です。たとえば、インポートはプラグイン変換の結果として作成される可能性があります。これは、Vite が最初のスキャンでインポートを検出できないことを意味します。つまり、ファイルがブラウザによって要求されて変換された後にのみ、インポートを検出できます。 これにより、サーバの起動後すぐにサーバが再バンドルされます。

これには、`include` と `exclude` の両方が使用できます。依存関係が大きい（多くの内部モジュールがある）場合や、CommonJS の場合には、それを含める必要があります。依存関係が小さく、すでに有効な ESM の場合には、それを除外し、ブラウザに直接読み込ませることができます。

## キャッシュ

### File System キャッシュ

Vite は、`node_modules/.vite` に、バンドル済みの依存関係をキャッシュします。これにより、いくつかのソースに基づいて、バンドル前のステップを再実行する必要があるかどうか:

- `package.json` の `dependencies` リスト。
- パッケージマネージャのロックファイル　例： `package-lock.json`、`yarn.lock`、または `pnpm-lock.yaml`。
- vite.config.js に関連するフィールドがあれば、それを入力してください。

上記のいずれかが変更された場合のみ、バンドル前のステップを再実行する必要があります。

何らかの理由で Vite に外れた場合の再バンドルを強制したい場合は、開発サーバを `--force` コマンドラインオプションで起動するか、手動で `node_modules/.vite` のキャッシュディレクトリを削除します。

### ブラウザ キャッシュ

解決された依存関係のリクエストは、開発中のページの再読み込みのパフォーマンスを向上させるために、HTTP ヘッダ `max-age = 31536000、immutable` で積極的にキャッシュされます。キャッシュされると、これらのリクエストは開発サーバに再びヒットすることはありません。異なるバージョンがインストールされている（パッケージマネージャのロックファイルに反映されている）場合は、追加されたバージョンクエリによって自動的に無効になります。 デバッグしたい場合ローカルで編集することで、依存関係を作成できます:

1. ブラウザの devtools のネットワークタブからキャッシュを一時的に無効にします。
2. Vite 開発サーバを `--force` フラグで再起動して、deps を再バンドルします。
3. ページをリロードします。
