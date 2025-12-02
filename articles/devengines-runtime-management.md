---
title: "パッケージマネージャーで Runtime のバージョンを管理する"
emoji: "⚙️"
type: "tech"
topics: ["nodejs", "npm", "pnpm", "yarn"]
published: true
published_at: "2025-12-03 09:00"
---

## はじめに

Node.js のバージョン管理には、従来から nvm や nodenv といったツールが使われてきました。しかし最近では、パッケージマネージャー自体が Node.js や Deno などの Runtime のバージョンを管理する方法が登場しています。

この記事では、OpenJS Foundation で標準化が進められている `devEngines` という仕組みを紹介し、npm、pnpm、yarn の各パッケージマネージャーにおける対応状況について解説します。

## devEngines とは

`devEngines` は package.json で開発環境の Runtime のバージョンを指定し、パッケージマネージャーが自動的にダウンロード・インストールする仕組みです。OpenJS Foundation の Package Metadata Interoperability Collab Space で[提案](https://github.com/openjs-foundation/package-metadata-interoperability-collab-space/issues/15)され、主に npm のメンテナによって策定されました。

通常のアプリケーション開発では、この `devEngines` を使用して開発環境の Runtime のバージョンを管理します。

### 従来の engines フィールドとの違い

従来から存在する `engines` フィールド（[npm](https://docs.npmjs.com/cli/v11/configuring-npm/package-json#engines) / [pnpm](https://pnpm.io/package_json#engines) / [yarn](https://classic.yarnpkg.com/lang/en/docs/package-json/#toc-engines)）との主な違いは以下の通りです。

- **engines**: パッケージの利用者が満たすべき環境要件（ライブラリ公開時に使用）
- **devEngines**: パッケージの開発者が使用する環境要件（アプリケーション開発時に使用）

```json
{
  "engines": {
    "node": ">=22.0.0"
  },
  "devEngines": {
    "runtime": {
      "name": "node",
      "version": ">=24.0.0",
      "onFail": "download"
    }
  }
}
```

### devEngines の仕様

[devEngines Field Proposal](https://github.com/openjs-foundation/package-metadata-interoperability-collab-space/blob/main/devengines-field-proposal.md) では以下のキーをサポートしています。

- **runtime**: JavaScript runtime（Node.js、Deno、Bun など）
- **packageManager**: パッケージマネージャー（npm、pnpm、yarn など）
- **cpu**: CPU アーキテクチャ
- **os**: オペレーティングシステム
- **libc**: C 標準ライブラリ

この記事では、**runtime** フィールドに焦点を当てて解説します。

### onFail オプション

`onFail` オプションで、バージョン要件を満たさない場合の動作を指定できます。

- **ignore**: 何もしない
- **warn**: 警告を表示して続行
- **error**: エラーを表示して終了（仕様上のデフォルト）
- **download**: 指定されたバージョンを自動ダウンロード

```json
{
  "devEngines": {
    "runtime": {
      "name": "node",
      "version": ">=24.0.0",
      "onFail": "download"
    }
  }
}
```

`onFail: "download"` を指定すると、要件を満たすバージョンが存在しない場合、パッケージマネージャーが自動的にダウンロードを試みます。これにより、nvm や nodenv などの追加ツールなしで Runtime のバージョン管理が可能になります。

:::message alert
**注意:** `onFail: "download"` は `devEngines` 仕様で定義されていますが、実際に対応しているかどうかは各パッケージマネージャーの実装によります。詳しくは次のセクションを参照してください。
:::

## パッケージマネージャーごとの対応状況

| パッケージマネージャー | devEngines 対応 | onFail: ignore | onFail: warn | onFail: error | onFail: download | 参考リンク |
|---|---|---|---|---|---|---|
| **npm** | 対応 (v10.9.0~) | 対応 | 対応 | 対応 | 未実装 | [PR #7766](https://github.com/npm/cli/pull/7766) |
| **pnpm** | 対応 (v10.14~) | 未実装 | 未実装 | 未実装 | 対応 | [PR #9755](https://github.com/pnpm/pnpm/pull/9755) |
| **yarn** | 未対応 | - | - | - | - | |

### 各パッケージマネージャーの詳細

#### npm

[v10.9.0 で `devEngines` の対応が完了](https://github.com/npm/cli/releases/tag/v10.9.0)しましたが、`onFail: "download"` は未実装です。詳しい説明はないものの、npm の開発チームは[セキュリティ上の懸念](https://github.com/npm/cli/issues/8003#issuecomment-2565859327)を理由に、あえて実装を見送っているようです。

**利用可能なオプション:** `onFail: "ignore"`, `onFail: "warn"`, `onFail: "error"`

```json
{
  "devEngines": {
    "runtime": {
      "name": "node",
      "version": ">=24.0.0",
      "onFail": "error"
    }
  }
}
```

#### pnpm

[v10.14.0 で `devEngines` の対応が完了](https://github.com/pnpm/pnpm/releases/tag/v10.14.0)し、`onFail: "download"` による自動ダウンロード機能が利用可能です。

:::message
pnpm は `onFail: "download"` のみをサポートしており、`error`、`warn`、`ignore` は未実装です。これらを指定しても無視されます。
:::

**利用可能なオプション:** `onFail: "download"`

```json
{
  "devEngines": {
    "runtime": {
      "name": "node",
      "version": ">=24.0.0",
      "onFail": "download"
    }
  }
}
```

#### yarn

yarn は [devEngines のサポート予定がありません](https://github.com/nodejs/corepack/issues/687#issuecomment-3025544220)。従来の `engines` フィールドでバージョン要件を指定することは可能ですが、自動インストールには対応していません。

## まとめ

`devEngines` は OpenJS Foundation で標準化が進められている、package.json で開発環境の runtime バージョンを管理する仕組みです。pnpm では `onFail: "download"` に対応しており、自動ダウンロードが可能です。npm は検証機能のみ、yarn は未対応という状況です。

Runtime のバージョン管理に悩んでいる方は、npm や pnpm が提供する `devEngines` を検討しても良いかもしれません。
