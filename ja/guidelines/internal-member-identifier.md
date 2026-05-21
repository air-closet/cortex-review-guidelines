# 社内メンバー identifier 共通ガイドライン

社内メンバー（社員）を引数として扱う MCP ツール / API / バッチ等に共通して適用する identifier
取扱いルール。Issue [#665](https://github.com/air-closet/cortex/issues/665) で導入。

## ルール

社内メンバーを identifier として受け取る処理では、次の 2 点を満たす必要がある。

1. **email と nickname のどちらでも受け取れるようにする**
   - 呼び出し側が手元に持っている情報で identity を指定できる
   - email 専用 / nickname 専用の片方しか受け付けない設計は禁止
2. **Google Workspace 同期データでメンバー identity を解決する**
   - 受け取った email / nickname を Google Workspace ディレクトリ（cortex では Firestore
     `(default)/users` コレクションが SoT）と突合する
   - 該当ユーザーが存在しない / 退職済み / 停止中の場合はエラー
   - 突合結果から `(email, nickname, displayName)` を確定し、以降の処理ではマスタ由来の値だけを使う

## なぜか

呼び出し側が任意の文字列で identity を持ち込めると、本来のユーザー identity との乖離が生じる。
例: 本来 `kei` のユーザーを `k.terayama` として登録できてしまうと、後から表記揺れを直すには
`remove_member` → `add_member` の破壊的やり直しが必要になり、退職や改名にも追従できない。
identity の単一情報源を Google Workspace に固定することで、表記揺れ・誤登録・履歴の壊れを
原理的に防止する。

## cortex での標準実装

`apps/mcp/git-server/src/sandbox/member-directory.ts` の `resolveMemberIdentity(identifier)`
が cortex の標準実装。Firestore `(default)/users` コレクション（`apps/pipeline/workspace/`
が Google Workspace から日次同期する）を突き合わせる。

### 期待する戻り値

```ts
type ResolvedMemberIdentity = {
  email: string;       // 小文字化済み・マスタ値
  nickname: string;    // マスタ値（HR スプレッドシートまたは GWS 由来）
  displayName: string; // googleWorkspace.name.fullName 由来。空なら nickname にフォールバック
};
```

### エラー条件

- identifier が空
- `users` コレクションに該当メンバーが存在しない
- `googleWorkspace.archived` または `googleWorkspace.suspended`
- `status` が `active` 以外
- nickname 未登録（HR スプレッドシートの更新を促す）

### identifier の判定

`identifier.includes('@')` を email として扱う。それ以外は nickname として扱い、まず
`nickname`（小文字フォーマット）で照会し、空なら `nickName`（camelCase 後方互換）に
フォールバックする。

## ツールが従うべきパラメータ仕様

```text
add_member(project_id, identifier, role)   # email | nickname のどちらか
remove_member(project_id, identifier)
update_member(project_id, identifier, ...) # role 変更時も nickname をマスタで再同期
```

- 旧来の `user_email` + `nickname` を別パラメータで受け取る設計は禁止
- nickname を呼び出し側から持ち込んで上書きしない（マスタ値のみを書き込む）
- 退職済みメンバーの削除など、`users` での解決が不可能なケースが必要なツールは、まず
  自ツール側のメンバー一覧（例: project_members）を identifier で照合してから
  `resolveMemberIdentity` にフォールバックする（`apps/mcp/git-server/src/sandbox/project-tools.ts`
  の `resolveProjectMemberEmail` を参考にする）

## 適用対象

- `sandbox MCP` の `sandbox_project_add_member` / `update_member` / `remove_member`（本 PR で適用）
- 既存の他ツール群もこのガイドラインに沿って順次見直す（社内メンバーを identifier として
  扱う MCP ツール / API / バッチ全般）
