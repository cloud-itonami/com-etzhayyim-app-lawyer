# cloud-itonami/com-etzhayyim-app-lawyer

**`lawyer.etzhayyim.com` 弁護士ポータルの、etzhayyim ブランド版の凍結コピー。**
実体は 129 行の thin edge facade Worker 1 本 —— 判断は何もせず、`/xrpc/…` を
`dispatcher.etzhayyim.com` へ中継するだけ。**その dispatcher も、自分のホストも、
今日は DNS に無い**（下記「ホストの現在地」）。

名前が `com-etzhayyim-app-lawyer` としか言っていないので、まずここで名乗る ——
この repo は **同じ設計の 2 つのブランド系統のうち、etzhayyim 側の 1 つ**であって、
「この workspace の弁護士業務の正本」ではない。動いている事務所 OS は別の repo。

## 最近接 repo との境界

| repo | 役割 | こことの違い |
|---|---|---|
| [`cloud-itonami/lawyer`](https://github.com/cloud-itonami/lawyer) | **同じ設計の gftd.ai 系統**。`ai.gftd.apps.lawyer.*`、`lawyer.gftd.ai` | **Worker の実装はブランド文字列を除いて同一**（下記）。あちらは lexicon 6 本と python LangGraph を持つ。ここは持たない代わりに manifest と vitest ハーネスを持つ |
| [`cloud-itonami/lawfirm`](https://github.com/cloud-itonami/lawfirm) | **事務所 OS**。受任〜終結を弁護士の判断を必ず経由する形で記録する governed actor（`LawFirmAdvisor ⊣ LawFirmGovernor`） | あちらが**実装のある事業**。ここは中継しかしない facade |
| [`cloud-itonami/kaisya`](https://github.com/cloud-itonami/kaisya) | **会社側の画面**。何を抱えていて何が問題になっているか | client 向け。ここは attorney 向け |
| **ここ** | **凍結コピー**。etzhayyim monorepo から抽出した 25 ファイル | 上のどれでもない。新しい弁護士業務をここに足さない |

**新しく弁護士業務を起こすなら `cloud-itonami/lawfirm` の側。** ここは
「etzhayyim ブランドで何が宣言されていたか」を失わないために在る。

### 2 つの系統は Worker が同一

`cloud-itonami/lawyer/worker/src/app.ts` と、ここの
`appview/etzhayyim-wasm-lawyer-334bbd5f/src/app.ts` は**どちらも 129 行で、
差分はブランド文字列 6 箇所だけ**（実測 2026-08-12、`etzhayyim`/`gftd` を同じ
トークンに正規化して diff したところ 5 hunk・全てブランド行）:

| | ここ | `cloud-itonami/lawyer` |
|---|---|---|
| NSID | `com.etzhayyim.apps.lawyer.*` | `ai.gftd.apps.lawyer.*` |
| handle | `lawyer.etzhayyim.com` | `lawyer.gftd.ai` |
| dispatcher 既定 | `dispatcher.etzhayyim.com` | `dispatcher.gftd.ai` |
| tenant | `etzhayyim` | `gftdcojp` |

**lexicon（NSID の schema 正本）はここには無い。** 6 command の lexicon JSON は
`cloud-itonami/lawyer/lexicons/lawyer/` にあり、そこの `id` は
`ai.gftd.apps.lawyer.*` —— つまり**ここが中継している `com.etzhayyim.apps.lawyer.*`
の schema は、この repo の中には書かれていない**。`CLAUDE.md` が指している
`00-contracts/lexicons/…` は抽出前の monorepo のパスで、ここには存在しない。

## 中身（28 ファイル / 167 KB。うち 25 は抽出時のまま）

```
appview/etzhayyim-wasm-lawyer-334bbd5f/
  src/app.ts              129 行。**この repo で唯一動くもの**。import ゼロ
  test/lawyer.test.ts     placeholder 1 件（expect(true).toBe(true)）
  wrangler.jsonc          Worker 設定。alias 8 件は死んでいる（下記）
  kotodama.jsonld         actor manifest（capabilities / derive rules / space）
  svelte/                 5 view。**wrangler の build に繋がっていない**（下記）
CLAUDE.md                 設計文書。リンクは monorepo 時代のパスで切れている
PROJECT.jsonld            did:web / tier / governance
bootstrap.sh              DID mint + record 登録。`etzhayyim` CLI が要る（無い）
README.edn / migration.edn / NOTICE   抽出の由来
docs/operator-quickstart.md           最初に踏む手順（実測つき）
docs/adr/0001-…                       この repo の境界
```

### 動くもの — Worker facade

`src/app.ts` は **import が 1 つも無い**ので、Cloudflare も wrangler も無しに
Node からそのまま呼べる。手順は [`docs/operator-quickstart.md`](docs/operator-quickstart.md)。
実測した振る舞い（2026-08-12）:

| 入力 | 出力 |
|---|---|
| `/health` `/_worker/health` `/_app/meta` | 200。command 6 + sharedCommand 5 + graph 2 の一覧 |
| `/xrpc/com.etzhayyim.apps.{lawyer,lawfirm}.*` | dispatcher へ POST 転送。query を body に畳み、`firmDid` を注入し、`x-internal-trust` を付ける |
| 壊れた POST body | 400 `InvalidJson` |
| それ以外 | 404 `NotFound` |

**判断はここには無い。** matter も grant も ISCO-2611 承認ゲートも、`CLAUDE.md` が
書いているものは全部 dispatcher の向こう側（LangServer / RisingWave）の話で、
この repo には 1 行も無い。ここに在るのは中継と `firmDid` の注入だけ。

### 動かないもの（実測。直していない）

- **`wrangler.jsonc` の `alias` 8 件は解決しない。** 全部
  `/Users/junkawasaki/etzhayyim/etzhayyim-apps-etzhayyim/…` という**このマシンに
  存在しない絶対パス**を指している。ただし `src/app.ts` も svelte 側も
  `@etzhayyim/*` を import していないので、**壊れているのではなく死んでいる**
  （テンプレートから複写された残骸）。
- **svelte の 5 view は deploy されない。** `wrangler.jsonc` の `main` は
  `src/app.ts` で、`assets` / `site` binding は無い。この設定で deploy しても
  出るのは JSON facade だけ。svelte 側は lockfile も持たない。
- **`wrangler` が依存に無い**ので、`CLAUDE.md` の「Deploy」節（`etzhayyim deploy`）は
  この repo 単体では踏めない。
- **`bootstrap.sh` は `etzhayyim` CLI を要求して exit 1 で止まる**（実測）。
  fail-closed なので害は無いが、CLI の配布元はこの repo には書かれていない。

## ホストの現在地（2026-08-12 実測）

| host | 結果 |
|---|---|
| `lawyer.etzhayyim.com` | **NXDOMAIN**（1.1.1.1 / 8.8.8.8 とも） |
| `dispatcher.etzhayyim.com` | **NXDOMAIN** —— 中継先そのものが無い |
| `etzhayyim.com`（apex） | 200（Cloudflare） |
| `lawyer.gftd.ai`（もう一方の系統） | DNS は在るが **522**（origin 到達不可） |

**したがって `PROJECT.jsonld` / `kotodama.jsonld` が名乗る
`did:web:lawyer.etzhayyim.com` は今日 resolve できない** —— did:web は
`https://lawyer.etzhayyim.com/.well-known/did.json` を引く定義なので、
DNS が無い時点で解決不能。manifest の DID を「登録済みの identity」として
引用しないこと。

## ここでやらないこと

- **新しい弁護士業務をここに足さない。** 実装のある事務所 OS は
  `cloud-itonami/lawfirm`（`docs/adr/0001-lawyer-facade-boundary.md`）。
- **`wrangler.jsonc` の alias を「直さない」。** 参照している import が無いので、
  直す先が無い。消すなら deploy 経路を復活させる作業と一緒にやる。
- **`README.edn` / `migration.edn` を書き換えない。** 抽出時の由来を固定した
  canonical EDN（`:allowed-additions ["README.edn" "migration.edn"]`）。
- **`CLAUDE.md` のリンクを辿らない。** `../../00-contracts/…` `60-apps/…` は
  抽出前の monorepo のパスで、この repo には存在しない。設計の記述としては
  読めるが、パスとしては全部切れている。
