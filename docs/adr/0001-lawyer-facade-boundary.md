# ADR-0001: この repo の役割と、弁護士業務を名乗る隣接 repo との境界

## Status

Accepted (2026-08-12)

## Context

この workspace には弁護士・法律事務所を名乗る repo が複数ある。実測
（`nbb scripts/repo-search.cljs lawfirm saiban bengoshi`、2026-08-12）:

- `cloud-itonami/lawfirm` — 受任〜終結を弁護士の判断を必ず経由する形で記録・監査する
  事務所 OS。langgraph StateGraph の governed actor（`LawFirmAdvisor ⊣ LawFirmGovernor`）
- `cloud-itonami/kaisya` — 会社側の画面（何を抱えていて何が問題になっているか）
- `cloud-itonami/lawyer` — attorney-facing portal。`ai.gftd.apps.lawyer.*` / `lawyer.gftd.ai`
- `cloud-itonami/com-etzhayyim-app-lawyer` — **この repo**。`com.etzhayyim.apps.lawyer.*` /
  `lawyer.etzhayyim.com`

この repo は README.md を持っておらず、名前も `com-etzhayyim-app-lawyer` としか
言っていなかった。そのため中身を見ないと `cloud-itonami/lawyer` との区別がつかず、
成熟度計測は `:repo/kind "unclassified"` として扱っていた。

### 実測して分かったこと（2026-08-12）

**1. `cloud-itonami/lawyer` とはブランド違いの同一実装である。**
両者の `src/app.ts` はどちらも 129 行で、`etzhayyim` / `gftd` を同じトークンに
正規化して diff すると **5 hunk・全てブランド文字列**（NSID 名前空間・handle・
dispatcher 既定 URL・tenant）。ロジックの差は無い。

非対称なのは周辺だけ:

| | ここ | `cloud-itonami/lawyer` |
|---|---|---|
| lexicon（NSID schema） | **無い** | `lexicons/lawyer/` に 6 本（`id` は `ai.gftd.apps.lawyer.*`） |
| LangGraph 実装 | 無い | `python/langgraph/` に 2 graph |
| actor manifest | `PROJECT.jsonld` + `kotodama.jsonld`（derive rules / space / capabilities） | `magatama.jsonld` のみ |
| test harness | vitest（中身は placeholder 1 件） | 無い |

つまり**ここが中継している `com.etzhayyim.apps.lawyer.*` の schema 正本は、
この repo の中には無い**。`CLAUDE.md` が指す `00-contracts/lexicons/…` は抽出前の
monorepo のパスで、この repo には存在しない。

**2. 両系統とも live ではない。**

| host | 実測 |
|---|---|
| `lawyer.etzhayyim.com` | NXDOMAIN（1.1.1.1 / 8.8.8.8） |
| `dispatcher.etzhayyim.com`（中継先） | NXDOMAIN |
| `lawyer.gftd.ai` | DNS は在るが 522（origin 到達不可） |

`PROJECT.jsonld` / `kotodama.jsonld` が名乗る `did:web:lawyer.etzhayyim.com` は、
did:web の定義上 `https://lawyer.etzhayyim.com/.well-known/did.json` を引くので、
**今日 resolve できない**。

**3. deploy 経路は抽出で切れている。** `wrangler` は依存に無く、`wrangler.jsonc` は
この repo に無い Cloudflare 資源（`secrets_store_secrets` 22・`hyperdrive`・
`services` 4）を要求する。`alias` 8 件は
`/Users/junkawasaki/etzhayyim/etzhayyim-apps-etzhayyim/…` という存在しない絶対パスを
指すが、**`src/app.ts` も svelte 側も `@etzhayyim/*` を import していない**ので
壊れているのではなく死んでいる（テンプレートからの複写）。`bootstrap.sh` は
`etzhayyim` CLI を要求して exit 1 で fail-closed する。

**4. それでも facade 本体は今日も動く。** `src/app.ts` は import ゼロなので、
Cloudflare も wrangler も無しに Node から `fetch(req, env)` を直接呼べる。
`docs/operator-quickstart.md` の step 3 が 4 ケース（meta 200 / 未知 404 /
xrpc の firmDid 注入と `x-internal-trust` 付与 / 壊れた JSON 400）を実測している。

## Decision

1. **この repo は etzhayyim ブランド版の凍結 facade であり、事業ではない。**
   README.md 冒頭でそう名乗り、上記 3 repo との境界を表で示す。名前が役割を
   示さない repo は README 冒頭で名乗る、という workspace 規則（ADR-2608040100 /
   concept 索引）の適用。
2. **新しい弁護士業務をここに足さない。** 実装のある事務所 OS は
   `cloud-itonami/lawfirm`。ここは「etzhayyim ブランドで何が宣言されていたか」を
   失わないために在る。
3. **`did:web:lawyer.etzhayyim.com` を「登録済みの identity」として引用しない。**
   manifest に書かれていることと resolve できることは別。
4. **死んだ `alias` / `wrangler.jsonc` をこの周では直さない。** 参照している import が
   無いので直す先が無い。撤去するなら deploy 経路の復活と同じ作業単位でやる。
5. **`README.edn` / `migration.edn` は書き換えない。** `migration.edn` の
   `:identity :allowed-additions ["README.edn" "migration.edn"]` が抽出時の由来を
   固定している。

## Consequences

- この repo を見た者が、`cloud-itonami/lawyer` との重複を「どちらかが古い」と
  誤読して片方を消す、という事故が起きにくくなる。**どちらも live ではなく、
  どちらも同じ実装で、非対称なのは周辺資産**という事実が README と ここに残る。
- 2 系統が同一実装のまま並存する状態自体は解消していない。workspace 規則
  （ADR-2608040100）の「同一面の同一主題は不可」に照らすと、いずれ統合するか
  片方を明示的に origin/role 面へ寄せる判断が要る。**この ADR はその判断をしない** ——
  両者とも live でない今、統合の受け皿（どちらの NSID を残すか）を決める材料が
  無いため。決めるのは `lawfirm` 側の事業が host を持ったとき。
- `docs/operator-quickstart.md` の step 3 は、そのまま vitest のテストに移せる形で
  書いてある（現在の `test/lawyer.test.ts` は `expect(true).toBe(true)` のみで
  `src/app.ts` を import していない）。**この ADR では test を足していない** ——
  1 反復 1 軸の規律に従い、この周は docs 軸だけを動かした。

## Alternatives considered

- **`cloud-itonami/lawyer` へ統合して この repo を tombstone にする。** 却下 ——
  こちらにしか無い資産（`kotodama.jsonld` の derive rules / space 定義、
  `PROJECT.jsonld` の governance、`bootstrap.sh` の DID mint 手順、vitest ハーネス）が
  失われる。統合するなら移送先を決めてからで、README を書く作業の副産物として
  やることではない。
- **死んだ `alias` と使えない `wrangler.jsonc` を掃除する。** 却下 —— 掃除しても
  deploy できるようにはならない（secret store も services も無い）。「動くように
  見えるが動かない」設定を作るだけで、現状より悪い。
