# app-mailer（経路判定）— itonami（dapp）上の分散型設計

状態: 第 1 スライス実装（2026-10-20）。正本はこの文書。
追跡: Linear JUN-128（親 JUN-116）。

## 位置づけ

app-mailer は mailer.etzhayyim.com の面 —— dispatcher / PDS / direct read の
経路分岐を持つ XRPC/API プロキシ（businessLogic は kotodama の mailer.py、
BPMN 契約）。本スライスはその**経路分岐の判定**を判定コア化。

## このスライス: 経路判定コア

| 層 | 実体 | 状態 |
|---|---|---|
| 判定コア guest | `aiueos/mailer_route.cljk` — admission（`ai.etzhayyim.apps.mailer.` プレフィックス **かつ** GET/POST のときだけ mailer 経路、他は PDS / health・listBindings は direct read / JSON 不正は 400）/ **fallback-ok（平文 fallback 禁止 = fail closed）** / query-merge（body 優先） | **実装済み・実測** |
| 記述子 | `aiueos/mailer-route.edn` | **実装済み** |
| parity | `svelte/test/mailer/route_parity_test.cljk` — 正本ソースの規則一致検査（プレフィックス / GET||POST / 400 InvalidJson / fallback 空 / query 補完条件） | **実装済み・実測** |

実測: `amu test aiueos/mailer_route.cljk` → **21/21**（jvm-kir / js / wasm）。
parity → 5 assertions / 0 failures。

## 正本の正確な規則（guest はこれに追従）

- **fallback は持たない**: dispatcher は公衆 `dispatcher.etzhayyim.com` のみ。
  平文 dev IP（sslip.io 等）へのフォールバックは撤去済み ——「primary が
  一時的 CF エラーを返すと生の error 520 を客に返していた」ことへの対策。
  guest の fallback-ok はこれを fail closed で判定。
- プレフィックス外の NSID や別メソッド（DELETE 等）は**エラーではなく
  PDS 経路**へ分岐。JSON 不正のみ 400 拒否。
- POST の空 body は {}。query は body に無いキーだけ補う（body 優先）。

## 実測メモ（parity の境界）

正本 TS は `$lib` / `@sveltejs/kit` 別名依存で kbb/node 直接実行不可。
parity は**ソース一致検査**（規則が変化したら落ちる drift 検出）で、
実行 parity は正本の `mailer.test.ts`（vitest）が担う構成。

## 未実装境界

- dispatcher 転送の実体（`x-internal-trust` ヘッダ、失敗時のエラー集約）は
  host（ネットワーク）。
- sendEmail 等の業務判定（kotodama mailer.py / BPMN）は別スライス候補。
- wasm 面（inbound-ml / notify / resend）の判定。
- mon（DNA）化、aiueos session 載せ（fleet 共通）。
