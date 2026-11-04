# Agent Errors

https://agenterrors.org

AI エージェントのハーネス層のエラーを標準化する Internet-Draft です。

`draft-ouchi-agent-harness-errors.md` は kramdown-rfc 形式の Internet-Draft です。

## ビルド

```sh
gem install kramdown-rfc
pip install xml2rfc
kdrfc -h draft-ouchi-agent-harness-errors.md   # .xml / .txt / .html を生成
```

## 提出の流れ

1. 提出前にファイル名と `docname` を `draft-ouchi-agent-harness-errors-00` にする
2. https://datatracker.ietf.org/submit/ から個人ドラフトとして提出する
3. 関連するメーリングリスト（HTTPAPI WG、エージェント関連の BoF や WG など）に紹介メールを送る
4. MCP 側では SEP（Specification Enhancement Proposal）、A2A 側では extension の提案を並行して出すと、`_meta` キーや extension URI を確定させやすい

まず martinthomson/internet-draft-template で GitHub リポジトリを作っておくと、CI で I-D を自動ビルドできます。

## agenterrors.org で公開が必要な URI

ドラフトの中で次の URI と識別子を使っています。URI のほうは、実際にアクセスできるようにしておく必要があります。

| URI / 識別子 | 用途 |
|---|---|
| `https://agenterrors.org/problems/harness-error` | IANA に登録されるまでの problem type URI |
| `https://agenterrors.org/ext/a2a/v1` | A2A の extension URI（A2A バインディングの説明ページ） |
| `org.agenterrors/harness-error` | MCP の `_meta` キー（URL ではないのでページは不要） |
