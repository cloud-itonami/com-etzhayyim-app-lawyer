# operator quickstart

この repo は「動いているサービス」ではなく**凍結コピー**なので、最初にやることは
起動ではなく **「何が今日も動き、何が死んでいるか」を自分の手で確かめること**。
所要 5 分、Cloudflare アカウントも wrangler も secret も要らない。

ここに書いてある手順は **2026-08-12 に実際に踏んで、貼ってある出力はその実測値**。
踏めなかった手順は step 5 に「踏めない」と書いてある。

## 0. 前提

Node だけ。実測した版:

```bash
node --version   # v26.3.0
npm --version    # 11.16.0
```

**step 3 は Node が `.ts` をそのまま import できることに依存する**（型注釈の
strip）。この repo の devDependencies に `tsx` / `ts-node` は無いので、古い Node
では step 3 だけ動かない。step 1・2・4 は影響を受けない。

作業ディレクトリは Worker のある場所:

```bash
cd appview/etzhayyim-wasm-lawyer-334bbd5f
```

## 1. 依存を入れる

```bash
npm ci
```

```
added 65 packages, and audited 66 packages in 9s
2 high severity vulnerabilities
```

これが `node_modules/` を作る。**この repo には `.gitignore` が無かった**ので
手順どおり踏むと 65 package が untracked で現れていた（`git add -A` 事故のもと）。
2026-08-12 に `.gitignore` を足したので、今は無視される。

**上の 2 件（nanoid / postcss）は vitest → vite 経由の dev 依存**で、
`npm audit` が名指しする通り devDependencies 側にしか居ない。**deploy される
artifact には入らない** —— `src/app.ts` は import ゼロで、runtime 依存が 1 つも無い
（step 3 で自分で確かめられる）。`npm audit fix` を急ぐ理由は無いが、
放置していることは知っておくこと。

## 2. Worker が型として通ることを確かめる

```bash
npm run typecheck      # = tsc --noEmit
```

出力なし・exit 0。`tsconfig.json` の `include` は `src/**/*.ts` なので、
**これが見ているのは Worker 本体だけ**（svelte 側も test 側も見ていない）。

## 3. facade を実際に呼ぶ（この repo の本体）

`src/app.ts` は **import が 1 つも無い**ので、Cloudflare 無しに Node から直接
`fetch(req, env)` を呼べる。dispatcher の代わりに**受け取った内容をそのまま返す
stub** を立てて、facade が何を上流へ渡しているかを見る:

```bash
node --input-type=module -e '
import http from "node:http";
const app = (await import("./src/app.ts")).default;
const srv = http.createServer((req,res)=>{let b="";req.on("data",c=>b+=c);req.on("end",()=>{
  res.writeHead(200,{"content-type":"application/json"});
  res.end(JSON.stringify({sawPath:req.url, sawBody:JSON.parse(b||"{}"),
                          sawTrust:req.headers["x-internal-trust"]??null}));});});
await new Promise(r=>srv.listen(0,r));
const env = {DISPATCHER_URL:`http://127.0.0.1:${srv.address().port}`,
             DISPATCHER_INTERNAL_SECRET:"stub-secret"};
const show = async (l,req,e=env)=>{const r=await app.fetch(req,e);
  console.log(l, r.status, (await r.text()).slice(0,160));};
await show("meta    ", new Request("https://x/_app/meta"), {});
await show("404     ", new Request("https://x/nope"), {});
await show("xrpc GET", new Request("https://x/xrpc/com.etzhayyim.apps.lawyer.getDashboard?matterId=m1"));
await show("badjson ", new Request("https://x/xrpc/com.etzhayyim.apps.lawyer.acceptGrant",
                                   {method:"POST",body:"{oops"}));
srv.close();'
```

```
meta     200 {"ok":true,"nanoid":"334bbd5f","handle":"lawyer.etzhayyim.com","tenant":"etzhayyim","firmDid":"did:web:lawyer.etzhayyim.com","lawyerDid":"","execution":"edge-la
404      404 {"error":"NotFound","message":"lawyer: endpoint not found"}
xrpc GET 200 {"sawPath":"/xrpc/com.etzhayyim.apps.lawyer.getDashboard","sawBody":{"matterId":"m1","firmDid":"did:web:lawyer.etzhayyim.com"},"sawTrust":"stub-secret"}
badjson  400 {"error":"InvalidJson"}
```

4 行が意味しているもの:

1. **meta は env 無し（`{}`）でも 200 を返す** —— 既定の firmDid
   `did:web:lawyer.etzhayyim.com` はコードに焼かれている
2. 未知のパスは 404 で止まる（proxy に落ちない）
3. **facade の仕事は 3 つだけ**: query param を body に畳む・`firmDid` を注入する・
   `x-internal-trust` を付けて転送する。`matterId=m1` が body に移り、
   宣言していない `firmDid` が足されているのが見える
4. 壊れた JSON は上流へ行かず 400 で止まる

**この 4 行が通っても「弁護士ポータルが動いている」ことにはならない。**
中継先の `dispatcher.etzhayyim.com` は step 6 のとおり DNS に無い。ここで
確かめたのは**中継の形**だけで、その先には何も無い。

## 4. テストを回す（今日は何も証明しない）

```bash
npm test               # = vitest run
```

```
 Test Files  1 passed (1)
      Tests  1 passed (1)
   Duration  384ms
```

**緑だが空。** `test/lawyer.test.ts` の中身は `expect(true).toBe(true)` 1 件だけで、
`src/app.ts` を import してすらいない。**この緑を「Worker が正しい」と読まないこと。**
今日 Worker の振る舞いを見る唯一の方法は step 3。

（step 3 の 4 ケースはそのまま vitest のテストにできる。まだ誰もやっていない。）

## 5. 踏めない手順（踏もうとして止まった記録）

- **deploy できない。** `wrangler` は devDependencies に無く、`wrangler.jsonc` は
  この repo に無い Cloudflare 資源（`secrets_store_secrets` 22 件・`hyperdrive`・
  4 つの `services` binding）を要求する。`AGENTS.md` の `etzhayyim deploy` は
  monorepo 時代の CLI 経由の手順。
- **`bootstrap.sh` は止まる。** 実測:

  ```bash
  bash bootstrap.sh phase1   # → error: etzhayyim CLI not found / exit 1
  bash bootstrap.sh          # → usage / exit 2
  ```

  **fail-closed なので害は無い**（DID mint も record 書き込みも始まらない）。
  ただし `${HOME}/.local/etzhayyim/bootstrap/lawyer-<date>.json` という**空の台帳
  ファイルだけは先に作られる**ので、消しておくこと。`etzhayyim` CLI の入手先は
  この repo に書かれていない。
- **svelte を build しても deploy 先が無い。** `wrangler.jsonc` の `main` は
  `src/app.ts` で `assets` / `site` binding が無いので、5 view はこの設定の
  出荷物に入らない。svelte 側は lockfile も無い（`npm install` は解決を毎回やり直す）。
  共有マシンなので、もし試すなら resource governor を通すこと:
  `node <superproject>/scripts/resource-guard.mjs run build -- npm run build`

## 6. ホストの生死を測る

```bash
dig +short @1.1.1.1 lawyer.etzhayyim.com        # (空) = NXDOMAIN
dig +short @1.1.1.1 dispatcher.etzhayyim.com    # (空) = NXDOMAIN
curl -s -o /dev/null -w '%{http_code}\n' https://lawyer.gftd.ai/_app/meta   # 522
```

`lawyer.etzhayyim.com` も中継先の `dispatcher.etzhayyim.com` も **DNS に無い**。
もう一方の系統（`cloud-itonami/lawyer` の `lawyer.gftd.ai`）は DNS こそ在るが
**522 = origin 到達不可**。

**つまり両系統とも今日は live ではない。** manifest の
`did:web:lawyer.etzhayyim.com` も、did:web の定義上
`https://lawyer.etzhayyim.com/.well-known/did.json` を引けないので resolve できない。
この DID を「登録済みの identity」として引用しないこと。

## 7. 次に読むもの

- [`../README.md`](../README.md) — この repo が何で、隣の 3 repo と何が違うか
- [`adr/0001-lawyer-facade-boundary.md`](adr/0001-lawyer-facade-boundary.md) — なぜ
  ここに弁護士業務を足さないか
- `AGENTS.md` — 設計の記述としては読める。**リンクは全部切れている**
  （`../../00-contracts/…` `60-apps/…` は抽出前の monorepo のパス）
