# Hermes-TS M3「上企业微信」实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 agent 通过企业微信自建应用与用户对话——手机上发消息、收到回复；验证 gateway + ChannelPort 平台无关架构。

**Architecture:** 六边形架构落地平台层：kernel 新增 `ChannelPort` 端口与 `InboundMessage`/`OutboundMessage` 中立消息模型；新包 `@hermes-ts/gateway`（平台无关的会话键、SessionManager 并发状态机、Router）与 `@hermes-ts/channel-wecom`（WXBizMsgCrypt 加解密、access_token 客户端、回调 HTTP 服务器、ChannelPort 实现）。CLI 新增 `gateway` 子命令作为组合根。

**Tech Stack:** node:http（零依赖回调服务器）、node:crypto（AES-256-CBC/SHA1）、既有 monorepo 工具链。企业微信自建应用（corpid+agentid+secret+回调 Token+EncodingAESKey）。

**Spec:** hermes-agent 仓库 `docs/superpowers/specs/2026-07-04-hermes-ts-rewrite-design.md` 第 7 节（gateway 与数据流）
**领域来源:** 旧 Python 版 `plugins/platforms/wecom/{callback_adapter.py, wecom_crypto.py}`（协议以其为准，行为移植非逐行翻译）
**前置:** M0+M1+M2 完成（tag v0.2.0-m2）；M2 backlog 的并发 idx 竞态在本计划 Task 1 关闭

## Global Constraints

- 仓库：`/Users/lirichen/Work/GithubRepo/hermes-ts`（直接在 main 上继续）
- TypeScript strict，全 ESM（.js import 后缀）；新包 `@hermes-ts/gateway`、`@hermes-ts/channel-wecom` 与现有包同构（private、exports "./src/index.ts"）
- **每个 Task 结束必须 `pnpm typecheck && pnpm lint && pnpm test` 全绿后才 commit**；conventional commits
- kernel 保持零运行时依赖；gateway 只依赖 kernel；channel-wecom 依赖 kernel + gateway，用 node 内建（http/crypto），**不引入 web 框架**
- **秘钥只从 env 读**：`WECOM_CORP_ID`、`WECOM_CORP_SECRET`、`WECOM_AGENT_ID`、`WECOM_CALLBACK_TOKEN`、`WECOM_ENCODING_AES_KEY`；绝不进 config.yaml schema。非秘钥开关（回调 host/port/path、启用平台）进 config schema
- 企业微信硬约束（协议以旧 Python `wecom_crypto.py` 为准）：
  - **收消息只能回调**（无轮询）；回调体上限 **65536 字节**，超限 413 且不解析
  - **WXBizMsgCrypt**：`msg_signature = SHA1(sort([token, timestamp, nonce, encrypt]) 拼接).hexdigest`；AES-256-CBC，key = `base64decode(EncodingAESKey + "=")`（32 字节），iv = key 前 16 字节；PKCS7 块大小 **32**；明文 = 16 随机字节 + 4 字节网络序长度 + XML + receiveid，receiveid 必须等于 corpid
  - **不能编辑已发消息** → UX：回调立即回 `"success"`（空 200），先主动推"收到，处理中…"回执，agent 跑完再主动推最终答案（>2048 字节自动分段）
  - **access_token**：`gettoken?corpid=&corpsecret=`，缓存，7200s 过期，取用时留 60s 安全边界；发送遇 errcode **40001/42001** 淘汰缓存并**重试一次**
  - **dedup**：按 MsgId，TTL **300s**（企业微信对超时/非 200 回调最多重试 3 次）；重复直接回 `"success"` 不重复入队
  - 发送用 `message/send` 的 **text** 类型（v1 纯文本，不用 markdown），2048 字节分段
- 时间戳 `new Date().toISOString()`；ID 用 `node:crypto` 的 `randomUUID()`；HMAC/hash 用 `node:crypto`

---

## Task 1: storage migration 002 — 并发 idx 唯一约束（关闭 M2 backlog）

**Files:**
- Modify: `packages/storage/src/schema.ts`, `packages/storage/src/sqlite-storage.ts`
- Test: `packages/storage/src/sqlite-storage.test.ts`（追加）

**Interfaces:**
- Produces: `MIGRATIONS` 增加 version 2（`CREATE UNIQUE INDEX ux_messages_session_idx ON messages(session_id, idx)`）；appendMessages 的 `MAX(idx)+1` 计算移入 `.immediate()` 事务内，消除跨事务竞态窗口

- [ ] **Step 1: 写失败测试**

`packages/storage/src/sqlite-storage.test.ts` 追加:
```ts
import Database from "better-sqlite3";
import { applyMigrations } from "./migrations.js";
import { MIGRATIONS } from "./schema.js";

describe("SqliteStorage concurrency safety (migration 002)", () => {
  it("enforces unique (session_id, idx)", async () => {
    const storage = new SqliteStorage(":memory:");
    const s = await storage.createSession("t");
    await storage.appendMessages(s.id, [userMessage("a")]);
    // 直接对底层库插入重复 idx 应被唯一索引拒绝
    const raw = new Database(":memory:");
    applyMigrations(raw, MIGRATIONS);
    raw.prepare("INSERT INTO sessions (id,title,created_at,updated_at,touch_seq) VALUES ('x','x','t','t',1)").run();
    const ins = raw.prepare(
      "INSERT INTO messages (session_id, idx, content_json, text_content, created_at) VALUES (?,?,?,?,?)",
    );
    ins.run("x", 0, "{}", "", "t");
    expect(() => ins.run("x", 0, "{}", "", "t")).toThrow(/UNIQUE/);
    raw.close();
    storage.close();
  });

  it("applies migration 2 on a fresh db (ux index present)", () => {
    const raw = new Database(":memory:");
    expect(applyMigrations(raw, MIGRATIONS)).toBeGreaterThanOrEqual(2);
    const idx = raw
      .prepare("SELECT name FROM sqlite_master WHERE type='index' AND name='ux_messages_session_idx'")
      .get();
    expect(idx).toBeDefined();
    raw.close();
  });
});
```

Run: `pnpm test` → 期望 FAIL（migration 2 不存在，唯一索引缺失）。

- [ ] **Step 2: 加 migration 002**

`packages/storage/src/schema.ts` 的 `MIGRATIONS` 数组追加：
```ts
  {
    version: 2,
    sql: `CREATE UNIQUE INDEX ux_messages_session_idx ON messages(session_id, idx);`,
  },
```

- [ ] **Step 3: idx 计算移入 immediate 事务**

`packages/storage/src/sqlite-storage.ts` 的 `appendMessages` 改为把 `nextIdx` 查询放进事务回调内，并用 immediate 事务（获取写锁再算 idx）：
```ts
  async appendMessages(sessionId: string, messages: ModelMessage[]): Promise<void> {
    const session = await this.getSession(sessionId);
    if (!session) throw new Error(`unknown session: ${sessionId}`);
    const now = new Date().toISOString();
    const insert = this.db.prepare(
      "INSERT INTO messages (session_id, idx, content_json, text_content, created_at) VALUES (?, ?, ?, ?, ?)",
    );
    const tx = this.db.transaction((batch: ModelMessage[]) => {
      const nextIdxRow = this.db
        .prepare("SELECT COALESCE(MAX(idx), -1) + 1 AS next FROM messages WHERE session_id = ?")
        .get(sessionId) as { next: number };
      let idx = nextIdxRow.next;
      for (const message of batch) {
        insert.run(sessionId, idx, JSON.stringify(message), messageTextOf(message), now);
        idx++;
      }
      this.db.prepare("UPDATE sessions SET updated_at = ?, touch_seq = (SELECT COALESCE(MAX(touch_seq),0)+1 FROM sessions) WHERE id = ?").run(now, sessionId);
    });
    tx.immediate(messages);
  }
```
（保持 touch_seq 单调 bump 逻辑不变，仅把 idx 计算与写入包进同一 immediate 事务。若原实现的 touch_seq 更新语句形态不同，保留原语句，只把 SELECT MAX(idx) 挪进事务并改用 `.immediate()`。）

- [ ] **Step 4: 验证并提交**

Run: `pnpm typecheck && pnpm lint && pnpm test` → PASS（既有 storage 测试 + 2 个新测试全绿）。

```bash
git add packages/storage && git commit -m "feat(storage): unique (session_id,idx) index and immediate-tx append (migration 002)"
```

---

## Task 2: kernel — ChannelPort 端口 + 中立消息模型

**Files:**
- Modify: `packages/kernel/src/ports.ts`, `packages/kernel/src/index.ts`
- Create: `packages/kernel/src/channel.ts`
- Test: `packages/kernel/src/channel.test.ts`

**Interfaces:**
- Produces（gateway 与 channel 适配器逐字依赖）:
  - `type InboundMessage = { channel: string; conversationId: string; senderId: string; text: string; messageId: string }`
  - `type OutboundMessage = { conversationId: string; text: string }`
  - `interface ChannelPort { readonly channel: string; start(handler: (msg: InboundMessage) => Promise<void>): Promise<void>; send(msg: OutboundMessage): Promise<void>; stop(): Promise<void> }`
  - `function inboundMessage(fields: {...}): InboundMessage`（构造 + trim text）

- [ ] **Step 1: 写失败测试**

`packages/kernel/src/channel.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import { inboundMessage } from "./channel.js";

describe("inboundMessage", () => {
  it("builds a normalized inbound message and trims text", () => {
    expect(
      inboundMessage({
        channel: "wecom",
        conversationId: "corp:u1",
        senderId: "u1",
        text: "  hi  ",
        messageId: "m1",
      }),
    ).toEqual({
      channel: "wecom",
      conversationId: "corp:u1",
      senderId: "u1",
      text: "hi",
      messageId: "m1",
    });
  });
});
```

Run: `pnpm test` → 期望 FAIL。

- [ ] **Step 2: 实现**

`packages/kernel/src/channel.ts`:
```ts
export type InboundMessage = {
  channel: string;
  conversationId: string;
  senderId: string;
  text: string;
  messageId: string;
};

export type OutboundMessage = {
  conversationId: string;
  text: string;
};

export interface ChannelPort {
  readonly channel: string;
  start(handler: (msg: InboundMessage) => Promise<void>): Promise<void>;
  send(msg: OutboundMessage): Promise<void>;
  stop(): Promise<void>;
}

export function inboundMessage(fields: InboundMessage): InboundMessage {
  return { ...fields, text: fields.text.trim() };
}
```

`packages/kernel/src/index.ts` 追加：
```ts
export * from "./channel.js";
```

（`ChannelPort` 定义在 channel.ts 而非 ports.ts，与 messages/events 平行；ports.ts 保留 Provider/Tool/Storage。）

- [ ] **Step 3: 验证并提交**

Run: `pnpm typecheck && pnpm lint && pnpm test` → PASS。

```bash
git add packages/kernel && git commit -m "feat(kernel): ChannelPort and neutral inbound/outbound message model"
```

---

## Task 3: channel-wecom — WXBizMsgCrypt 加解密（round-trip 测试）

**Files:**
- Create: `packages/channel-wecom/package.json`, `packages/channel-wecom/tsconfig.json`, `packages/channel-wecom/src/index.ts`, `packages/channel-wecom/src/crypto.ts`
- Test: `packages/channel-wecom/src/crypto.test.ts`

**Interfaces:**
- Produces:
  - `class WecomCrypto { constructor(opts: { token: string; encodingAesKey: string; receiveId: string }); signature(timestamp: string, nonce: string, encrypt: string): string; verify(msgSignature: string, timestamp: string, nonce: string, encrypt: string): boolean; decrypt(encrypt: string): string; encrypt(plaintext: string): string }`
  - decrypt 校验 receiveId，不符抛 `Error("receive_id mismatch")`；verify 用时间恒定比较

- [ ] **Step 1: 建包**

`packages/channel-wecom/package.json`:
```json
{
  "name": "@hermes-ts/channel-wecom",
  "version": "0.0.1",
  "private": true,
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "dependencies": {
    "@hermes-ts/kernel": "workspace:*",
    "@hermes-ts/gateway": "workspace:*"
  }
}
```
（gateway 依赖在 Task 6 前不会被 crypto 用到，但先声明；若 pnpm 因 gateway 尚不存在报错，先只留 kernel 依赖，Task 6 建 gateway 后补上。）

`packages/channel-wecom/tsconfig.json`:
```json
{ "extends": "../../tsconfig.base.json", "include": ["src"] }
```

`pnpm install`（若 gateway 未建导致 workspace 解析失败，先从 package.json 移除 gateway 依赖，Task 9 再加回）。

- [ ] **Step 2: 写失败测试（round-trip + 签名 + receiveId）**

`packages/channel-wecom/src/crypto.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import { WecomCrypto } from "./crypto.js";

// EncodingAESKey 必须是 43 字符（base64 无填充，解码后 32 字节）
const AES_KEY = "abcdefghijklmnopqrstuvwxyz0123456789ABCDEFG"; // 43 chars
const opts = { token: "tok", encodingAesKey: AES_KEY, receiveId: "corp123" };

describe("WecomCrypto", () => {
  it("round-trips encrypt → decrypt", () => {
    const crypto = new WecomCrypto(opts);
    const xml = "<xml><Content><![CDATA[你好 hello]]></Content></xml>";
    const encrypted = crypto.encrypt(xml);
    expect(crypto.decrypt(encrypted)).toBe(xml);
  });

  it("signature is order-independent over the four fields and deterministic", () => {
    const crypto = new WecomCrypto(opts);
    const sig = crypto.signature("1700000000", "nonce1", "ENC");
    expect(sig).toMatch(/^[0-9a-f]{40}$/);
    expect(crypto.signature("1700000000", "nonce1", "ENC")).toBe(sig);
  });

  it("verify accepts a correct signature and rejects a wrong one", () => {
    const crypto = new WecomCrypto(opts);
    const enc = crypto.encrypt("<xml></xml>");
    const good = crypto.signature("t", "n", enc);
    expect(crypto.verify(good, "t", "n", enc)).toBe(true);
    expect(crypto.verify("0".repeat(40), "t", "n", enc)).toBe(false);
  });

  it("decrypt rejects a mismatched receiveId", () => {
    const a = new WecomCrypto(opts);
    const b = new WecomCrypto({ ...opts, receiveId: "otherCorp" });
    const enc = a.encrypt("<xml></xml>");
    expect(() => b.decrypt(enc)).toThrow(/receive_id mismatch/);
  });

  it("rejects an EncodingAESKey of wrong length", () => {
    expect(() => new WecomCrypto({ ...opts, encodingAesKey: "short" })).toThrow(/43/);
  });
});
```

Run: `pnpm test` → 期望 FAIL。

- [ ] **Step 3: 实现**

`packages/channel-wecom/src/crypto.ts`:
```ts
import { createCipheriv, createDecipheriv, createHash, randomBytes, timingSafeEqual } from "node:crypto";

const BLOCK_SIZE = 32;

function pkcs7Pad(buf: Buffer): Buffer {
  const pad = BLOCK_SIZE - (buf.length % BLOCK_SIZE);
  return Buffer.concat([buf, Buffer.alloc(pad, pad)]);
}

function pkcs7Unpad(buf: Buffer): Buffer {
  const pad = buf[buf.length - 1] ?? 0;
  if (pad < 1 || pad > BLOCK_SIZE) return buf;
  return buf.subarray(0, buf.length - pad);
}

export class WecomCrypto {
  private readonly token: string;
  private readonly key: Buffer;
  private readonly iv: Buffer;
  private readonly receiveId: string;

  constructor(opts: { token: string; encodingAesKey: string; receiveId: string }) {
    if (opts.encodingAesKey.length !== 43) {
      throw new Error("EncodingAESKey must be 43 characters");
    }
    this.token = opts.token;
    this.key = Buffer.from(`${opts.encodingAesKey}=`, "base64");
    this.iv = this.key.subarray(0, 16);
    this.receiveId = opts.receiveId;
  }

  signature(timestamp: string, nonce: string, encrypt: string): string {
    const joined = [this.token, timestamp, nonce, encrypt].sort().join("");
    return createHash("sha1").update(joined).digest("hex");
  }

  verify(msgSignature: string, timestamp: string, nonce: string, encrypt: string): boolean {
    const expected = this.signature(timestamp, nonce, encrypt);
    if (expected.length !== msgSignature.length) return false;
    return timingSafeEqual(Buffer.from(expected), Buffer.from(msgSignature));
  }

  encrypt(plaintext: string): string {
    const text = Buffer.from(plaintext, "utf8");
    const random16 = randomBytes(16);
    const lenBuf = Buffer.alloc(4);
    lenBuf.writeUInt32BE(text.length, 0); // 网络序（大端）
    const raw = Buffer.concat([random16, lenBuf, text, Buffer.from(this.receiveId, "utf8")]);
    const cipher = createCipheriv("aes-256-cbc", this.key, this.iv);
    cipher.setAutoPadding(false);
    return Buffer.concat([cipher.update(pkcs7Pad(raw)), cipher.final()]).toString("base64");
  }

  decrypt(encrypt: string): string {
    const decipher = createDecipheriv("aes-256-cbc", this.key, this.iv);
    decipher.setAutoPadding(false);
    const decrypted = pkcs7Unpad(
      Buffer.concat([decipher.update(Buffer.from(encrypt, "base64")), decipher.final()]),
    );
    const content = decrypted.subarray(16); // 去掉 16 随机字节
    const msgLen = content.readUInt32BE(0);
    const xml = content.subarray(4, 4 + msgLen).toString("utf8");
    const receiveId = content.subarray(4 + msgLen).toString("utf8");
    if (receiveId !== this.receiveId) throw new Error("receive_id mismatch");
    return xml;
  }
}
```

`packages/channel-wecom/src/index.ts`:
```ts
export * from "./crypto.js";
```

- [ ] **Step 4: 验证并提交**

Run: `pnpm typecheck && pnpm lint && pnpm test` → PASS。

```bash
git add packages/channel-wecom pnpm-lock.yaml pnpm-workspace.yaml && git commit -m "feat(channel-wecom): WXBizMsgCrypt AES-256-CBC crypto with round-trip tests"
```

---

## Task 4: channel-wecom — access_token 客户端

**Files:**
- Create: `packages/channel-wecom/src/token.ts`
- Modify: `packages/channel-wecom/src/index.ts`
- Test: `packages/channel-wecom/src/token.test.ts`

**Interfaces:**
- Produces:
  - `type FetchLike = (url: string) => Promise<{ json(): Promise<unknown> }>`
  - `class AccessTokenClient { constructor(opts: { corpId: string; corpSecret: string; now?: () => number; fetch?: FetchLike }); get(): Promise<string>; invalidate(): void }`
  - 缓存 token，`expires_at > now + 60_000` 才复用；`get()` 首次/过期时请求 `gettoken`；`invalidate()` 清缓存（发送遇 40001/42001 时调用后重试）

- [ ] **Step 1: 写失败测试**

`packages/channel-wecom/src/token.test.ts`:
```ts
import { describe, expect, it, vi } from "vitest";
import { AccessTokenClient } from "./token.js";

function fakeFetch(responses: unknown[]) {
  let i = 0;
  const calls: string[] = [];
  const fetch = async (url: string) => {
    calls.push(url);
    const body = responses[Math.min(i++, responses.length - 1)];
    return { json: async () => body };
  };
  return { fetch, calls };
}

describe("AccessTokenClient", () => {
  it("fetches once and caches within the TTL", async () => {
    const { fetch, calls } = fakeFetch([{ errcode: 0, access_token: "T1", expires_in: 7200 }]);
    const client = new AccessTokenClient({ corpId: "c", corpSecret: "s", now: () => 0, fetch });
    expect(await client.get()).toBe("T1");
    expect(await client.get()).toBe("T1");
    expect(calls).toHaveLength(1);
    expect(calls[0]).toContain("corpid=c");
    expect(calls[0]).toContain("corpsecret=s");
  });

  it("refreshes after invalidate()", async () => {
    const { fetch, calls } = fakeFetch([
      { errcode: 0, access_token: "T1", expires_in: 7200 },
      { errcode: 0, access_token: "T2", expires_in: 7200 },
    ]);
    const client = new AccessTokenClient({ corpId: "c", corpSecret: "s", now: () => 0, fetch });
    expect(await client.get()).toBe("T1");
    client.invalidate();
    expect(await client.get()).toBe("T2");
    expect(calls).toHaveLength(2);
  });

  it("throws a readable error on non-zero errcode", async () => {
    const { fetch } = fakeFetch([{ errcode: 40013, errmsg: "invalid corpid" }]);
    const client = new AccessTokenClient({ corpId: "c", corpSecret: "s", now: () => 0, fetch });
    await expect(client.get()).rejects.toThrow(/40013/);
  });

  it("re-fetches once the cached token is within 60s of expiry", async () => {
    let t = 0;
    const { fetch, calls } = fakeFetch([
      { errcode: 0, access_token: "T1", expires_in: 100 },
      { errcode: 0, access_token: "T2", expires_in: 100 },
    ]);
    const client = new AccessTokenClient({ corpId: "c", corpSecret: "s", now: () => t, fetch });
    expect(await client.get()).toBe("T1"); // expires_at = 100_000
    t = 41_000; // 100_000 - 41_000 = 59_000 < 60_000 safety window → refresh
    expect(await client.get()).toBe("T2");
    expect(calls).toHaveLength(2);
  });
});
```

Run: `pnpm test` → 期望 FAIL。

- [ ] **Step 2: 实现**

`packages/channel-wecom/src/token.ts`:
```ts
export type FetchLike = (url: string) => Promise<{ json(): Promise<unknown> }>;

const GETTOKEN_URL = "https://qyapi.weixin.qq.com/cgi-bin/gettoken";
const SAFETY_MS = 60_000;

type TokenResponse = { errcode?: number; errmsg?: string; access_token?: string; expires_in?: number };

export class AccessTokenClient {
  private readonly corpId: string;
  private readonly corpSecret: string;
  private readonly now: () => number;
  private readonly fetchImpl: FetchLike;
  private cached: { token: string; expiresAt: number } | null = null;

  constructor(opts: { corpId: string; corpSecret: string; now?: () => number; fetch?: FetchLike }) {
    this.corpId = opts.corpId;
    this.corpSecret = opts.corpSecret;
    this.now = opts.now ?? Date.now;
    this.fetchImpl = opts.fetch ?? ((url) => fetch(url));
  }

  invalidate(): void {
    this.cached = null;
  }

  async get(): Promise<string> {
    if (this.cached && this.cached.expiresAt > this.now() + SAFETY_MS) {
      return this.cached.token;
    }
    const url = `${GETTOKEN_URL}?corpid=${encodeURIComponent(this.corpId)}&corpsecret=${encodeURIComponent(this.corpSecret)}`;
    const res = (await (await this.fetchImpl(url)).json()) as TokenResponse;
    if (res.errcode && res.errcode !== 0) {
      throw new Error(`gettoken failed: errcode ${res.errcode} ${res.errmsg ?? ""}`);
    }
    if (!res.access_token) throw new Error("gettoken returned no access_token");
    this.cached = {
      token: res.access_token,
      expiresAt: this.now() + (res.expires_in ?? 7200) * 1000,
    };
    return this.cached.token;
  }
}
```

`packages/channel-wecom/src/index.ts` 追加：
```ts
export * from "./token.js";
```

- [ ] **Step 3: 验证并提交**

Run: `pnpm typecheck && pnpm lint && pnpm test` → PASS。

```bash
git add packages/channel-wecom && git commit -m "feat(channel-wecom): access-token client with 60s-skew cache and invalidate"
```

---

## Task 5: channel-wecom — 入站 XML 解析 + 出站发送

**Files:**
- Create: `packages/channel-wecom/src/message.ts`
- Modify: `packages/channel-wecom/src/index.ts`
- Test: `packages/channel-wecom/src/message.test.ts`

**Interfaces:**
- Consumes: Task 4 的 `AccessTokenClient`
- Produces:
  - `type WecomInbound = { fromUser: string; msgType: string; content: string; msgId: string; agentId: string } | null`
  - `function parseInboundXml(xml: string): WecomInbound`（正则提取 CDATA，无 XML 解析器无 XXE；MsgType=event 且 Event∈{subscribe,enter_agent} → 返回 null 静默；其他 event 空 Content → content 置 "/start"；非 text/event → null）
  - `function chunkText(text: string, max = 2048): string[]`（按字节 2048 分段，优先换行边界；UTF-8 字节计）
  - `async function sendText(token: AccessTokenClient, agentId: string, toUser: string, text: string, fetch?: FetchLike): Promise<void>`（message/send text；40001/42001 → token.invalidate() 重试一次；分段逐条发）

- [ ] **Step 1: 写失败测试**

`packages/channel-wecom/src/message.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import { AccessTokenClient } from "./token.js";
import { chunkText, parseInboundXml, sendText } from "./message.js";

const TEXT_XML = `<xml><ToUserName><![CDATA[corp]]></ToUserName><FromUserName><![CDATA[u1]]></FromUserName><CreateTime>1700000000</CreateTime><MsgType><![CDATA[text]]></MsgType><Content><![CDATA[你好]]></Content><MsgId>12345</MsgId><AgentID>1000002</AgentID></xml>`;

describe("parseInboundXml", () => {
  it("extracts a text message", () => {
    expect(parseInboundXml(TEXT_XML)).toEqual({
      fromUser: "u1",
      msgType: "text",
      content: "你好",
      msgId: "12345",
      agentId: "1000002",
    });
  });

  it("silently drops subscribe/enter_agent events (returns null)", () => {
    const xml = `<xml><FromUserName><![CDATA[u1]]></FromUserName><MsgType><![CDATA[event]]></MsgType><Event><![CDATA[enter_agent]]></Event><MsgId>1</MsgId><AgentID>1</AgentID></xml>`;
    expect(parseInboundXml(xml)).toBeNull();
  });

  it("maps other empty-content events to /start", () => {
    const xml = `<xml><FromUserName><![CDATA[u1]]></FromUserName><MsgType><![CDATA[event]]></MsgType><Event><![CDATA[click]]></Event><MsgId>2</MsgId><AgentID>1</AgentID></xml>`;
    expect(parseInboundXml(xml)?.content).toBe("/start");
  });

  it("returns null for image/voice", () => {
    const xml = `<xml><FromUserName><![CDATA[u1]]></FromUserName><MsgType><![CDATA[image]]></MsgType><MsgId>3</MsgId><AgentID>1</AgentID></xml>`;
    expect(parseInboundXml(xml)).toBeNull();
  });
});

describe("chunkText", () => {
  it("returns a single chunk under the limit", () => {
    expect(chunkText("hello", 2048)).toEqual(["hello"]);
  });

  it("splits by byte length preferring newline boundaries", () => {
    const line = "x".repeat(1000);
    const chunks = chunkText(`${line}\n${line}\n${line}`, 2048);
    expect(chunks.length).toBeGreaterThan(1);
    for (const c of chunks) expect(Buffer.byteLength(c, "utf8")).toBeLessThanOrEqual(2048);
  });

  it("counts multibyte chars by UTF-8 bytes", () => {
    const chunks = chunkText("好".repeat(1000), 2048); // 3 bytes each = 3000 bytes
    expect(chunks.length).toBe(2);
  });
});

describe("sendText", () => {
  it("posts message/send and retries once on token-expired errcode", async () => {
    const bodies = [{ errcode: 42001, errmsg: "expired" }, { errcode: 0 }];
    const gets = [{ errcode: 0, access_token: "T1", expires_in: 7200 }, { errcode: 0, access_token: "T2", expires_in: 7200 }];
    const posted: string[] = [];
    let gi = 0;
    let pi = 0;
    const tokenFetch = async () => ({ json: async () => gets[Math.min(gi++, gets.length - 1)] });
    const token = new AccessTokenClient({ corpId: "c", corpSecret: "s", now: () => 0, fetch: tokenFetch });
    const sendFetch = async (url: string) => {
      posted.push(url);
      return { json: async () => bodies[Math.min(pi++, bodies.length - 1)] };
    };
    await sendText(token, "1000002", "u1", "hi", sendFetch as never);
    expect(posted).toHaveLength(2); // 首发 42001，刷新 token 后重试成功
    expect(posted[1]).toContain("access_token=T2");
  });
});
```

Run: `pnpm test` → 期望 FAIL。

- [ ] **Step 2: 实现**

`packages/channel-wecom/src/message.ts`:
```ts
import type { AccessTokenClient, FetchLike } from "./token.js";

const SEND_URL = "https://qyapi.weixin.qq.com/cgi-bin/message/send";
const SILENT_EVENTS = new Set(["subscribe", "enter_agent"]);

export type WecomInbound = {
  fromUser: string;
  msgType: string;
  content: string;
  msgId: string;
  agentId: string;
} | null;

function field(xml: string, tag: string): string {
  // 提取 <Tag><![CDATA[..]]></Tag> 或 <Tag>..</Tag>；无 XML 解析器 → 无 XXE
  const cdata = new RegExp(`<${tag}><!\\[CDATA\\[([\\s\\S]*?)\\]\\]></${tag}>`).exec(xml);
  if (cdata) return cdata[1] ?? "";
  const plain = new RegExp(`<${tag}>([\\s\\S]*?)</${tag}>`).exec(xml);
  return plain ? (plain[1] ?? "").trim() : "";
}

export function parseInboundXml(xml: string): WecomInbound {
  const msgType = field(xml, "MsgType");
  const base = {
    fromUser: field(xml, "FromUserName"),
    msgId: field(xml, "MsgId"),
    agentId: field(xml, "AgentID"),
  };
  if (msgType === "text") {
    return { ...base, msgType, content: field(xml, "Content") };
  }
  if (msgType === "event") {
    const event = field(xml, "Event");
    if (SILENT_EVENTS.has(event)) return null;
    return { ...base, msgType, content: field(xml, "Content") || "/start" };
  }
  return null; // image/voice/... 不在 v1 范围
}

export function chunkText(text: string, max = 2048): string[] {
  if (Buffer.byteLength(text, "utf8") <= max) return [text];
  const chunks: string[] = [];
  let current = "";
  const flush = () => {
    if (current) chunks.push(current);
    current = "";
  };
  for (const line of text.split("\n")) {
    const candidate = current ? `${current}\n${line}` : line;
    if (Buffer.byteLength(candidate, "utf8") <= max) {
      current = candidate;
      continue;
    }
    flush();
    if (Buffer.byteLength(line, "utf8") <= max) {
      current = line;
    } else {
      // 单行超限：按字节硬切（不切断多字节字符）
      let buf = "";
      for (const ch of line) {
        if (Buffer.byteLength(buf + ch, "utf8") > max) {
          chunks.push(buf);
          buf = ch;
        } else {
          buf += ch;
        }
      }
      current = buf;
    }
  }
  flush();
  return chunks;
}

export async function sendText(
  token: AccessTokenClient,
  agentId: string,
  toUser: string,
  text: string,
  fetchImpl: FetchLike = (url, init) => fetch(url, init) as never,
): Promise<void> {
  for (const chunk of chunkText(text)) {
    await sendChunk(token, agentId, toUser, chunk, fetchImpl, true);
  }
}

async function sendChunk(
  token: AccessTokenClient,
  agentId: string,
  toUser: string,
  content: string,
  fetchImpl: FetchLike,
  retry: boolean,
): Promise<void> {
  const accessToken = await token.get();
  const url = `${SEND_URL}?access_token=${encodeURIComponent(accessToken)}`;
  const body = {
    touser: toUser,
    msgtype: "text",
    agentid: Number(agentId),
    text: { content },
    safe: 0,
  };
  const res = (await (
    await fetchImpl(url, { method: "POST", body: JSON.stringify(body) } as never)
  ).json()) as { errcode?: number; errmsg?: string };
  if (res.errcode === 40001 || res.errcode === 42001) {
    if (retry) {
      token.invalidate();
      return sendChunk(token, agentId, toUser, content, fetchImpl, false);
    }
  }
  if (res.errcode && res.errcode !== 0) {
    throw new Error(`message/send failed: errcode ${res.errcode} ${res.errmsg ?? ""}`);
  }
}
```

注意：`FetchLike` 需扩展支持第二参 init（POST）。修改 `token.ts` 的 `FetchLike` 为 `(url: string, init?: unknown) => Promise<{ json(): Promise<unknown> }>` 并让 token 的 GET 调用忽略 init；测试里的 fake fetch 已兼容（多余参数忽略）。实施者据此微调 token.ts 的类型与 message.ts 的调用，保持 typecheck 通过。

`packages/channel-wecom/src/index.ts` 追加：
```ts
export * from "./message.js";
```

- [ ] **Step 3: 验证并提交**

Run: `pnpm typecheck && pnpm lint && pnpm test` → PASS。

```bash
git add packages/channel-wecom && git commit -m "feat(channel-wecom): inbound XML parse and message/send with byte-aware chunking"
```

---

## Task 6: gateway — 会话键（平台无关，纯函数）

**Files:**
- Create: `packages/gateway/package.json`, `packages/gateway/tsconfig.json`, `packages/gateway/src/index.ts`, `packages/gateway/src/session-key.ts`
- Test: `packages/gateway/src/session-key.test.ts`

**Interfaces:**
- Produces:
  - `function buildSessionKey(msg: { channel: string; conversationId: string }): string`（格式 `agent:main:<channel>:<conversationId>`；企业微信 conversationId 已是 `corp:user` 作用域键，天然按用户隔离）

- [ ] **Step 1: 建包 + 失败测试**

`packages/gateway/package.json`:
```json
{
  "name": "@hermes-ts/gateway",
  "version": "0.0.1",
  "private": true,
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "dependencies": {
    "@hermes-ts/kernel": "workspace:*"
  }
}
```
`packages/gateway/tsconfig.json`:
```json
{ "extends": "../../tsconfig.base.json", "include": ["src"] }
```
`pnpm install`

`packages/gateway/src/session-key.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import { buildSessionKey } from "./session-key.js";

describe("buildSessionKey", () => {
  it("namespaces by channel and conversation", () => {
    expect(buildSessionKey({ channel: "wecom", conversationId: "corp:u1" })).toBe(
      "agent:main:wecom:corp:u1",
    );
  });

  it("keeps distinct conversations distinct", () => {
    expect(buildSessionKey({ channel: "wecom", conversationId: "corp:u1" })).not.toBe(
      buildSessionKey({ channel: "wecom", conversationId: "corp:u2" }),
    );
  });
});
```

Run: `pnpm test` → FAIL。

- [ ] **Step 2: 实现**

`packages/gateway/src/session-key.ts`:
```ts
export function buildSessionKey(msg: { channel: string; conversationId: string }): string {
  return `agent:main:${msg.channel}:${msg.conversationId}`;
}
```
`packages/gateway/src/index.ts`:
```ts
export * from "./session-key.js";
```

- [ ] **Step 3: 验证并提交**

Run: `pnpm typecheck && pnpm lint && pnpm test` → PASS。

```bash
git add packages/gateway pnpm-lock.yaml pnpm-workspace.yaml && git commit -m "feat(gateway): platform-agnostic session key"
```

---

## Task 7: gateway — SessionManager 并发状态机（应用层串行化）

**Files:**
- Create: `packages/gateway/src/session-manager.ts`
- Modify: `packages/gateway/src/index.ts`
- Test: `packages/gateway/src/session-manager.test.ts`

**Interfaces:**
- Consumes: kernel `Agent`/`ModelMessage`/`AgentEvent`、`StoragePort`；gateway `buildSessionKey`
- Produces:
  - `type TurnResult = { text: string }`
  - `class SessionManager { constructor(deps: { storage: StoragePort; runTurn: (sessionKey: string, input: string, history: ModelMessage[]) => AsyncIterable<AgentEvent> }); submit(sessionKey: string, conversationTitle: string, input: string): Promise<TurnResult> }`
  - **每个 sessionKey 串行化**（同一 key 的 submit 排队，前一个完成后才跑下一个——关闭 M2 backlog 的应用层半边）；内部：载入/新建会话 → 收集事件流的文本 → 持久化增量 → 返回聚合文本

- [ ] **Step 1: 写失败测试**

`packages/gateway/src/session-manager.test.ts`:
```ts
import { InMemoryStorage, type AgentEvent, type ModelMessage } from "@hermes-ts/kernel";
import { describe, expect, it } from "vitest";
import { SessionManager } from "./session-manager.js";

function scriptedTurn(text: string, order: number[], marker: number): (k: string, i: string, h: ModelMessage[]) => AsyncIterable<AgentEvent> {
  return async function* () {
    order.push(marker);
    yield { type: "turn-start" } as AgentEvent;
    yield { type: "text-delta", text } as AgentEvent;
    yield {
      type: "turn-complete",
      messages: [{ role: "user", content: [{ type: "text", text: "in" }] }, { role: "assistant", content: [{ type: "text", text }] }],
    } as AgentEvent;
  };
}

describe("SessionManager", () => {
  it("runs a turn, returns aggregated text, and persists history", async () => {
    const storage = new InMemoryStorage();
    const sm = new SessionManager({ storage, runTurn: scriptedTurn("hello", [], 0) });
    const result = await sm.submit("k1", "chat", "hi");
    expect(result.text).toBe("hello");
    const [session] = await storage.listSessions();
    expect(session).toBeDefined();
    expect((await storage.loadMessages(session!.id)).at(-1)).toEqual({
      role: "assistant",
      content: [{ type: "text", text: "hello" }],
    });
  });

  it("serializes concurrent submits for the same session key", async () => {
    const storage = new InMemoryStorage();
    const order: number[] = [];
    let call = 0;
    const runTurn = () => {
      const marker = call++;
      return (async function* () {
        order.push(marker * 10 + 1); // start
        await new Promise((r) => setTimeout(r, marker === 0 ? 30 : 0));
        order.push(marker * 10 + 2); // end
        yield { type: "turn-complete", messages: [] } as AgentEvent;
      })();
    };
    const sm = new SessionManager({ storage, runTurn });
    await Promise.all([sm.submit("k", "c", "a"), sm.submit("k", "c", "b")]);
    // 第二个必须等第一个结束：0-start,0-end 在 1-start 之前
    expect(order).toEqual([1, 2, 11, 12]);
  });

  it("allows different session keys to run concurrently", async () => {
    const storage = new InMemoryStorage();
    const order: number[] = [];
    const runTurn = (key: string) =>
      (async function* () {
        order.push(Number(key === "k2"));
        await new Promise((r) => setTimeout(r, key === "k1" ? 20 : 0));
        yield { type: "turn-complete", messages: [] } as AgentEvent;
      })();
    const sm = new SessionManager({ storage, runTurn });
    await Promise.all([sm.submit("k1", "c", "a"), sm.submit("k2", "c", "b")]);
    expect(order[0]).toBe(0); // both started before k1 finished → interleaved
    expect(order).toContain(1);
  });
});
```

Run: `pnpm test` → FAIL。

- [ ] **Step 2: 实现**

`packages/gateway/src/session-manager.ts`:
```ts
import type { AgentEvent, ModelMessage, SessionMeta, StoragePort } from "@hermes-ts/kernel";

export type TurnResult = { text: string };

type RunTurn = (sessionKey: string, input: string, history: ModelMessage[]) => AsyncIterable<AgentEvent>;

export class SessionManager {
  private readonly storage: StoragePort;
  private readonly runTurn: RunTurn;
  private readonly chains = new Map<string, Promise<unknown>>();
  private readonly sessionIds = new Map<string, string>();

  constructor(deps: { storage: StoragePort; runTurn: RunTurn }) {
    this.storage = deps.storage;
    this.runTurn = deps.runTurn;
  }

  submit(sessionKey: string, conversationTitle: string, input: string): Promise<TurnResult> {
    const prior = this.chains.get(sessionKey) ?? Promise.resolve();
    const next = prior.then(
      () => this.runOne(sessionKey, conversationTitle, input),
      () => this.runOne(sessionKey, conversationTitle, input),
    );
    this.chains.set(
      sessionKey,
      next.catch(() => undefined),
    );
    return next;
  }

  private async runOne(sessionKey: string, title: string, input: string): Promise<TurnResult> {
    const session = await this.resolveSession(sessionKey, title);
    const history = await this.storage.loadMessages(session.id);
    let text = "";
    let finalMessages: ModelMessage[] = [];
    for await (const event of this.runTurn(sessionKey, input, history)) {
      if (event.type === "text-delta") text += event.text;
      else if (event.type === "turn-complete") finalMessages = event.messages;
    }
    const fresh = finalMessages.slice(history.length);
    if (fresh.length > 0) await this.storage.appendMessages(session.id, fresh);
    return { text };
  }

  private async resolveSession(sessionKey: string, title: string): Promise<SessionMeta> {
    const known = this.sessionIds.get(sessionKey);
    if (known) {
      const existing = await this.storage.getSession(known);
      if (existing) return existing;
    }
    const created = await this.storage.createSession(title);
    this.sessionIds.set(sessionKey, created.id);
    return created;
  }
}
```

（说明：sessionKey→sessionId 的内存映射让同一会话跨轮复用同一 storage session；进程重启后映射丢失会新建会话——v1 可接受，跨重启的会话恢复进 backlog。串行化用 per-key promise 链，`prior.then(run, run)` 保证前一轮无论成败都不阻塞后一轮，且严格排队。）

`packages/gateway/src/index.ts` 追加：
```ts
export * from "./session-manager.js";
```

- [ ] **Step 3: 验证并提交**

Run: `pnpm typecheck && pnpm lint && pnpm test` → PASS。

```bash
git add packages/gateway && git commit -m "feat(gateway): SessionManager with per-key serialization and persistence"
```

---

## Task 8: gateway — Router（UX：即时回执 + 最终答案）

**Files:**
- Create: `packages/gateway/src/router.ts`
- Modify: `packages/gateway/src/index.ts`
- Test: `packages/gateway/src/router.test.ts`

**Interfaces:**
- Consumes: kernel `ChannelPort`/`InboundMessage`；gateway `buildSessionKey`、`SessionManager`
- Produces:
  - `class Router { constructor(deps: { channel: ChannelPort; sessions: SessionManager; ackText?: string }); start(): Promise<void>; stop(): Promise<void> }`
  - 语义：channel.start 注册 handler → 每条 inbound：先 `channel.send(ackText)`（默认"收到，处理中…"）→ `sessions.submit(sessionKey, title, text)` → `channel.send(结果文本)`；submit 抛错时 send 一条错误提示（`[出错] <message>`）

- [ ] **Step 1: 写失败测试**

`packages/gateway/src/router.test.ts`:
```ts
import type { ChannelPort, InboundMessage, OutboundMessage } from "@hermes-ts/kernel";
import { InMemoryStorage, type AgentEvent } from "@hermes-ts/kernel";
import { describe, expect, it } from "vitest";
import { Router } from "./router.js";
import { SessionManager } from "./session-manager.js";

function fakeChannel() {
  const sent: OutboundMessage[] = [];
  let handler: ((m: InboundMessage) => Promise<void>) | null = null;
  const channel: ChannelPort = {
    channel: "wecom",
    start: async (h) => {
      handler = h;
    },
    send: async (m) => {
      sent.push(m);
    },
    stop: async () => {},
  };
  return { channel, sent, inject: (m: InboundMessage) => handler!(m) };
}

const inbound = (text: string): InboundMessage => ({
  channel: "wecom",
  conversationId: "corp:u1",
  senderId: "u1",
  text,
  messageId: "m1",
});

describe("Router", () => {
  it("acks immediately then sends the final answer", async () => {
    const { channel, sent, inject } = fakeChannel();
    const sm = new SessionManager({
      storage: new InMemoryStorage(),
      runTurn: () =>
        (async function* () {
          yield { type: "text-delta", text: "答案" } as AgentEvent;
          yield { type: "turn-complete", messages: [] } as AgentEvent;
        })(),
    });
    const router = new Router({ channel, sessions: sm });
    await router.start();
    await inject(inbound("问题"));
    expect(sent.map((s) => s.text)).toEqual(["收到，处理中…", "答案"]);
    expect(sent[0]?.conversationId).toBe("corp:u1");
  });

  it("surfaces turn errors to the user", async () => {
    const { channel, sent, inject } = fakeChannel();
    const sm = new SessionManager({
      storage: new InMemoryStorage(),
      runTurn: () =>
        (async function* () {
          throw new Error("boom");
        })(),
    });
    const router = new Router({ channel, sessions: sm });
    await router.start();
    await inject(inbound("问题"));
    expect(sent[0]?.text).toBe("收到，处理中…");
    expect(sent[1]?.text).toContain("boom");
  });
});
```

Run: `pnpm test` → FAIL。

- [ ] **Step 2: 实现**

`packages/gateway/src/router.ts`:
```ts
import type { ChannelPort, InboundMessage } from "@hermes-ts/kernel";
import { buildSessionKey } from "./session-key.js";
import type { SessionManager } from "./session-manager.js";

const DEFAULT_ACK = "收到，处理中…";

export class Router {
  private readonly channel: ChannelPort;
  private readonly sessions: SessionManager;
  private readonly ackText: string;

  constructor(deps: { channel: ChannelPort; sessions: SessionManager; ackText?: string }) {
    this.channel = deps.channel;
    this.sessions = deps.sessions;
    this.ackText = deps.ackText ?? DEFAULT_ACK;
  }

  async start(): Promise<void> {
    await this.channel.start((msg) => this.handle(msg));
  }

  async stop(): Promise<void> {
    await this.channel.stop();
  }

  private async handle(msg: InboundMessage): Promise<void> {
    await this.channel.send({ conversationId: msg.conversationId, text: this.ackText });
    const sessionKey = buildSessionKey(msg);
    try {
      const result = await this.sessions.submit(sessionKey, `${msg.channel} ${msg.conversationId}`, msg.text);
      await this.channel.send({ conversationId: msg.conversationId, text: result.text || "（无输出）" });
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      await this.channel.send({ conversationId: msg.conversationId, text: `[出错] ${message}` });
    }
  }
}
```

`packages/gateway/src/index.ts` 追加：
```ts
export * from "./router.js";
```

- [ ] **Step 3: 验证并提交**

Run: `pnpm typecheck && pnpm lint && pnpm test` → PASS。

```bash
git add packages/gateway && git commit -m "feat(gateway): Router with ack-then-answer UX and error surfacing"
```

---

## Task 9: channel-wecom — 回调 HTTP 服务器（ChannelPort 实现）

**Files:**
- Create: `packages/channel-wecom/src/adapter.ts`
- Modify: `packages/channel-wecom/src/index.ts`, `packages/channel-wecom/package.json`（确认 gateway 依赖存在）
- Test: `packages/channel-wecom/src/adapter.test.ts`

**Interfaces:**
- Consumes: kernel `ChannelPort`/`InboundMessage`/`inboundMessage`；本包 `WecomCrypto`/`AccessTokenClient`/`parseInboundXml`/`sendText`
- Produces:
  - `type WecomConfig = { corpId: string; corpSecret: string; agentId: string; token: string; encodingAesKey: string; host: string; port: number; path: string }`
  - `class WecomChannel implements ChannelPort`（constructor(config, opts?: { fetch?; dedupTtlMs? })）
  - 内部纯函数导出供测试：`function handleGet(crypto, query): { status: number; body: string }`（校验签名→解密 echostr→回明文；失败 403）；`async function handlePost(crypto, body, deps): { status: number; body: string }`（校验签名→解密→parseInboundXml→dedup→非重复则触发 handler（fire-and-forget）→回 "success"）

- [ ] **Step 1: 写失败测试（纯函数层，不起真实 socket）**

`packages/channel-wecom/src/adapter.test.ts`:
```ts
import { describe, expect, it, vi } from "vitest";
import { WecomCrypto } from "./crypto.js";
import { handleGet, handlePost } from "./adapter.js";

const AES_KEY = "abcdefghijklmnopqrstuvwxyz0123456789ABCDEFG";
const crypto = new WecomCrypto({ token: "tok", encodingAesKey: AES_KEY, receiveId: "corp" });

describe("handleGet (echostr verify)", () => {
  it("returns the decrypted echostr on a valid signature", () => {
    const echo = crypto.encrypt("<xml></xml>");
    const sig = crypto.signature("t", "n", echo);
    const res = handleGet(crypto, { msg_signature: sig, timestamp: "t", nonce: "n", echostr: echo });
    expect(res.status).toBe(200);
    expect(res.body).toBe("<xml></xml>");
  });

  it("403s on a bad signature", () => {
    const echo = crypto.encrypt("<xml></xml>");
    const res = handleGet(crypto, { msg_signature: "bad", timestamp: "t", nonce: "n", echostr: echo });
    expect(res.status).toBe(403);
  });
});

describe("handlePost", () => {
  const textXml = `<xml><FromUserName><![CDATA[u1]]></FromUserName><MsgType><![CDATA[text]]></MsgType><Content><![CDATA[hi]]></Content><MsgId>1</MsgId><AgentID>7</AgentID></xml>`;

  function envelope(inner: string, ts = "t", nonce = "n") {
    const enc = crypto.encrypt(inner);
    const body = `<xml><Encrypt><![CDATA[${enc}]]></Encrypt></xml>`;
    const sig = crypto.signature(ts, nonce, enc);
    return { body, query: { msg_signature: sig, timestamp: ts, nonce } };
  }

  it("decrypts, dispatches the inbound, and returns success", async () => {
    const { body, query } = envelope(textXml);
    const seen: string[] = [];
    const res = await handlePost(crypto, body, {
      query,
      onMessage: async (m) => {
        seen.push(m.text);
      },
      seen: new Map(),
      now: () => 0,
      dedupTtlMs: 300_000,
    });
    expect(res.status).toBe(200);
    expect(res.body).toBe("success");
    await new Promise((r) => setTimeout(r, 0)); // let fire-and-forget dispatch run
    expect(seen).toEqual(["hi"]);
  });

  it("dedups a repeated MsgId within the TTL", async () => {
    const { body, query } = envelope(textXml);
    const seen = new Map<string, number>();
    let count = 0;
    const deps = { query, onMessage: async () => { count++; }, seen, now: () => 0, dedupTtlMs: 300_000 };
    await handlePost(crypto, body, deps);
    await handlePost(crypto, body, deps);
    await new Promise((r) => setTimeout(r, 0));
    expect(count).toBe(1);
  });

  it("403s on a bad signature", async () => {
    const { body } = envelope(textXml);
    const res = await handlePost(crypto, body, {
      query: { msg_signature: "bad", timestamp: "t", nonce: "n" },
      onMessage: async () => {},
      seen: new Map(),
      now: () => 0,
      dedupTtlMs: 300_000,
    });
    expect(res.status).toBe(403);
  });
});
```

Run: `pnpm test` → FAIL。

- [ ] **Step 2: 实现（纯函数 + ChannelPort 壳）**

`packages/channel-wecom/src/adapter.ts`:
```ts
import { createServer, type Server } from "node:http";
import type { ChannelPort, InboundMessage, OutboundMessage } from "@hermes-ts/kernel";
import { inboundMessage } from "@hermes-ts/kernel";
import { WecomCrypto } from "./crypto.js";
import { parseInboundXml } from "./message.js";
import { AccessTokenClient, type FetchLike } from "./token.js";
import { sendText } from "./message.js";

export type WecomConfig = {
  corpId: string;
  corpSecret: string;
  agentId: string;
  token: string;
  encodingAesKey: string;
  host: string;
  port: number;
  path: string;
};

const MAX_BODY = 65_536;

function encryptField(xml: string): string {
  const m = /<Encrypt><!\[CDATA\[([\s\S]*?)\]\]><\/Encrypt>/.exec(xml);
  return m ? (m[1] ?? "") : "";
}

export function handleGet(
  crypto: WecomCrypto,
  query: { msg_signature?: string; timestamp?: string; nonce?: string; echostr?: string },
): { status: number; body: string } {
  const { msg_signature = "", timestamp = "", nonce = "", echostr = "" } = query;
  if (!crypto.verify(msg_signature, timestamp, nonce, echostr)) return { status: 403, body: "" };
  try {
    return { status: 200, body: crypto.decrypt(echostr) };
  } catch {
    return { status: 403, body: "" };
  }
}

export type PostDeps = {
  query: { msg_signature?: string; timestamp?: string; nonce?: string };
  onMessage: (msg: InboundMessage) => Promise<void>;
  seen: Map<string, number>;
  now: () => number;
  dedupTtlMs: number;
};

export async function handlePost(
  crypto: WecomCrypto,
  body: string,
  deps: PostDeps,
): Promise<{ status: number; body: string }> {
  const encrypt = encryptField(body);
  const { msg_signature = "", timestamp = "", nonce = "" } = deps.query;
  if (!crypto.verify(msg_signature, timestamp, nonce, encrypt)) return { status: 403, body: "" };
  let xml: string;
  try {
    xml = crypto.decrypt(encrypt);
  } catch {
    return { status: 403, body: "" };
  }
  const parsed = parseInboundXml(xml);
  if (!parsed) return { status: 200, body: "success" };

  // dedup（TTL 内重复 MsgId 直接成功）
  const now = deps.now();
  for (const [id, ts] of deps.seen) {
    if (now - ts > deps.dedupTtlMs) deps.seen.delete(id);
  }
  if (deps.seen.has(parsed.msgId)) return { status: 200, body: "success" };
  deps.seen.set(parsed.msgId, now);

  const msg = inboundMessage({
    channel: "wecom",
    conversationId: parsed.fromUser, // 纯函数层不知 corpId；corp:user 作用域前缀在 WecomChannel.start 的 onMessage 包装里补
    senderId: parsed.fromUser,
    text: parsed.content,
    messageId: parsed.msgId,
  });
  // fire-and-forget：立即回 success，处理异步进行（不阻塞 5s 回调窗口）
  void deps.onMessage(msg).catch(() => undefined);
  return { status: 200, body: "success" };
}

export class WecomChannel implements ChannelPort {
  readonly channel = "wecom";
  private readonly config: WecomConfig;
  private readonly crypto: WecomCrypto;
  private readonly token: AccessTokenClient;
  private readonly fetchImpl: FetchLike;
  private readonly seen = new Map<string, number>();
  private readonly dedupTtlMs: number;
  private server: Server | null = null;

  constructor(config: WecomConfig, opts: { fetch?: FetchLike; dedupTtlMs?: number } = {}) {
    this.config = config;
    this.crypto = new WecomCrypto({
      token: config.token,
      encodingAesKey: config.encodingAesKey,
      receiveId: config.corpId,
    });
    this.token = new AccessTokenClient({
      corpId: config.corpId,
      corpSecret: config.corpSecret,
      fetch: opts.fetch,
    });
    this.fetchImpl = opts.fetch ?? ((url, init) => fetch(url, init as never) as never);
    this.dedupTtlMs = opts.dedupTtlMs ?? 300_000;
  }

  async start(handler: (msg: InboundMessage) => Promise<void>): Promise<void> {
    // conversationId 作用域为 corp:user，避免多企业碰撞
    const onMessage = (msg: InboundMessage) =>
      handler({ ...msg, conversationId: `${this.config.corpId}:${msg.senderId}` });

    this.server = createServer((req, res) => {
      const url = new URL(req.url ?? "/", `http://${this.config.host}`);
      if (url.pathname !== this.config.path) {
        res.writeHead(404).end();
        return;
      }
      const query = Object.fromEntries(url.searchParams);
      if (req.method === "GET") {
        const out = handleGet(this.crypto, query);
        res.writeHead(out.status, { "content-type": "text/plain" }).end(out.body);
        return;
      }
      if (req.method === "POST") {
        let body = "";
        let tooLarge = false;
        req.on("data", (c) => {
          body += c;
          if (body.length > MAX_BODY) {
            tooLarge = true;
            res.writeHead(413).end();
            req.destroy();
          }
        });
        req.on("end", () => {
          if (tooLarge) return;
          void handlePost(this.crypto, body, {
            query,
            onMessage,
            seen: this.seen,
            now: () => Date.now(),
            dedupTtlMs: this.dedupTtlMs,
          }).then((out) => res.writeHead(out.status, { "content-type": "text/plain" }).end(out.body));
        });
        return;
      }
      res.writeHead(405).end();
    });

    await new Promise<void>((resolve) => this.server!.listen(this.config.port, this.config.host, resolve));
  }

  async send(msg: OutboundMessage): Promise<void> {
    const toUser = msg.conversationId.includes(":")
      ? msg.conversationId.slice(msg.conversationId.indexOf(":") + 1)
      : msg.conversationId;
    await sendText(this.token, this.config.agentId, toUser, msg.text, this.fetchImpl);
  }

  async stop(): Promise<void> {
    if (this.server) await new Promise<void>((resolve) => this.server!.close(() => resolve()));
    this.server = null;
  }
}
```

`packages/channel-wecom/src/index.ts` 追加：
```ts
export * from "./adapter.js";
```

确认 `packages/channel-wecom/package.json` 的 dependencies 含 `"@hermes-ts/gateway": "workspace:*"` 与 `"@hermes-ts/kernel": "workspace:*"`（gateway 现已存在），`pnpm install`。

- [ ] **Step 3: 验证并提交**

Run: `pnpm typecheck && pnpm lint && pnpm test` → PASS。

```bash
git add packages/channel-wecom pnpm-lock.yaml && git commit -m "feat(channel-wecom): WeCom callback server implementing ChannelPort"
```

---

## Task 10: config — gateway/wecom 配置节

**Files:**
- Modify: `packages/config/src/schema.ts`, `packages/config/src/loader.ts`
- Create: `packages/config/src/wecom.ts`
- Test: `packages/config/src/wecom.test.ts`

**Interfaces:**
- Produces:
  - schema 追加非秘钥节 `gateway?: { channel: "wecom"; callbackHost?: string; callbackPort?: number; callbackPath?: string }`（默认 host `0.0.0.0`、port `8645`、path `/wecom/callback`）
  - `function loadWecomConfig(env: Record<string,string|undefined>, gateway): WecomConfig`（从 env 读五个秘钥 WECOM_CORP_ID/CORP_SECRET/AGENT_ID/CALLBACK_TOKEN/ENCODING_AES_KEY，缺任一抛人类可读错误含变量名；合并 gateway 的 host/port/path 默认值）返回 channel-wecom 的 `WecomConfig` 形状

- [ ] **Step 1: 写失败测试**

`packages/config/src/wecom.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import { loadWecomConfig } from "./wecom.js";

const fullEnv = {
  WECOM_CORP_ID: "corp",
  WECOM_CORP_SECRET: "secret",
  WECOM_AGENT_ID: "1000002",
  WECOM_CALLBACK_TOKEN: "tok",
  WECOM_ENCODING_AES_KEY: "abcdefghijklmnopqrstuvwxyz0123456789ABCDEFG",
};

describe("loadWecomConfig", () => {
  it("reads secrets from env and applies gateway defaults", () => {
    expect(loadWecomConfig(fullEnv, {})).toEqual({
      corpId: "corp",
      corpSecret: "secret",
      agentId: "1000002",
      token: "tok",
      encodingAesKey: "abcdefghijklmnopqrstuvwxyz0123456789ABCDEFG",
      host: "0.0.0.0",
      port: 8645,
      path: "/wecom/callback",
    });
  });

  it("honors gateway overrides", () => {
    const cfg = loadWecomConfig(fullEnv, { callbackPort: 9000, callbackPath: "/cb" });
    expect(cfg.port).toBe(9000);
    expect(cfg.path).toBe("/cb");
  });

  it("throws naming the missing env var", () => {
    const { WECOM_CALLBACK_TOKEN, ...partial } = fullEnv;
    expect(() => loadWecomConfig(partial, {})).toThrow(/WECOM_CALLBACK_TOKEN/);
  });
});
```

Run: `pnpm test` → FAIL。

- [ ] **Step 2: 实现**

`packages/config/src/schema.ts` 的 configSchema 追加字段（在 `.strict()` 之前）：
```ts
    gateway: z
      .object({
        channel: z.literal("wecom"),
        callbackHost: z.string().default("0.0.0.0"),
        callbackPort: z.number().int().min(1).max(65535).default(8645),
        callbackPath: z.string().default("/wecom/callback"),
      })
      .strict()
      .optional(),
```

`packages/config/src/wecom.ts`:
```ts
export type WecomRuntimeConfig = {
  corpId: string;
  corpSecret: string;
  agentId: string;
  token: string;
  encodingAesKey: string;
  host: string;
  port: number;
  path: string;
};

function required(env: Record<string, string | undefined>, key: string): string {
  const value = env[key];
  if (!value) throw new Error(`${key} 未设置（企业微信自建应用需要它）`);
  return value;
}

export function loadWecomConfig(
  env: Record<string, string | undefined>,
  gateway: { callbackHost?: string; callbackPort?: number; callbackPath?: string },
): WecomRuntimeConfig {
  return {
    corpId: required(env, "WECOM_CORP_ID"),
    corpSecret: required(env, "WECOM_CORP_SECRET"),
    agentId: required(env, "WECOM_AGENT_ID"),
    token: required(env, "WECOM_CALLBACK_TOKEN"),
    encodingAesKey: required(env, "WECOM_ENCODING_AES_KEY"),
    host: gateway.callbackHost ?? "0.0.0.0",
    port: gateway.callbackPort ?? 8645,
    path: gateway.callbackPath ?? "/wecom/callback",
  };
}
```

`packages/config/src/index.ts` 追加 `export * from "./wecom.js";`。

- [ ] **Step 3: 验证并提交**

Run: `pnpm typecheck && pnpm lint && pnpm test` → PASS。

```bash
git add packages/config && git commit -m "feat(config): gateway/wecom config section with env-only secrets"
```

---

## Task 11: cli — `gateway` 子命令（组合根）

**Files:**
- Create: `apps/cli/src/gateway-main.ts`
- Modify: `apps/cli/src/main.ts`, `apps/cli/package.json`（deps 加 gateway、channel-wecom）, `README.md`
- Test: `apps/cli/src/gateway-main.test.ts`

**Interfaces:**
- Consumes: 全部——config、storage、providers、tools、kernel Agent、gateway Router/SessionManager、channel-wecom WecomChannel
- Produces:
  - `function buildGateway(deps): Router`（纯组装函数，注入 storage/provider/tools/config，返回未 start 的 Router，供测试）
  - `main.ts`：`process.argv[2] === "gateway"` 时走 gateway 模式（起 Router、打印回调 URL 提示、SIGINT 优雅停机），否则原 REPL 不变

- [ ] **Step 1: 写失败测试（buildGateway 纯组装，不起 socket）**

`apps/cli/src/gateway-main.test.ts`:
```ts
import { InMemoryStorage, type ChannelPort } from "@hermes-ts/kernel";
import { describe, expect, it } from "vitest";
import { buildGateway } from "./gateway-main.js";

function fakeChannel(): ChannelPort {
  return { channel: "wecom", start: async () => {}, send: async () => {}, stop: async () => {} };
}

describe("buildGateway", () => {
  it("wires a Router over the given channel + storage + agent config", () => {
    const router = buildGateway({
      channel: fakeChannel(),
      storage: new InMemoryStorage(),
      agentConfig: { model: "m", systemPrompt: "s", maxIterations: 10 },
      provider: { stream: async function* () { yield { type: "finish", stopReason: "stop" }; } } as never,
      tools: { definitions: () => [], execute: async () => ({ output: "", isError: false }) },
    });
    expect(router).toBeDefined();
    expect(typeof router.start).toBe("function");
  });
});
```

Run: `pnpm test` → FAIL。

- [ ] **Step 2: 实现**

`apps/cli/src/gateway-main.ts`:
```ts
import { Agent, type AgentConfig, type ChannelPort, type ProviderPort, type StoragePort, type ToolPort } from "@hermes-ts/kernel";
import { Router, SessionManager } from "@hermes-ts/gateway";

export function buildGateway(deps: {
  channel: ChannelPort;
  storage: StoragePort;
  agentConfig: AgentConfig;
  provider: ProviderPort;
  tools: ToolPort;
}): Router {
  const agent = new Agent(deps.agentConfig, { provider: deps.provider, tools: deps.tools });
  const sessions = new SessionManager({
    storage: deps.storage,
    runTurn: (_key, input, history) => agent.runTurn(input, history),
  });
  return new Router({ channel: deps.channel, sessions });
}
```

`apps/cli/src/main.ts`：在文件顶部分流（gateway 模式 vs REPL）。抽出一个 `runGateway()`：
```ts
async function runGateway(): Promise<void> {
  const config = await loadConfig();
  if (config.gateway?.channel !== "wecom") {
    console.error("config.gateway.channel 必须为 'wecom' 才能启动 gateway");
    process.exitCode = 1;
    return;
  }
  const provider = createProvider(config, process.env);
  await mkdir(dirname(config.storagePath), { recursive: true });
  const storage = new SqliteStorage(config.storagePath);
  const tools = createToolRegistry([shellTool, readFileTool, writeFileTool], { cwd: process.cwd() });
  const wecom = loadWecomConfig(process.env, config.gateway);
  const channel = new WecomChannel(wecom);
  const router = buildGateway({
    channel,
    storage,
    agentConfig: { model: config.model, systemPrompt: config.systemPrompt, maxIterations: config.maxIterations },
    provider,
    tools,
  });
  await router.start();
  console.log(`企业微信 gateway 已启动：回调监听 http://${wecom.host}:${wecom.port}${wecom.path}`);
  console.log("在企业微信管理后台把「接收消息服务器URL」指向该地址的公网映射。Ctrl+C 停止。");
  const shutdown = async () => {
    await router.stop();
    storage.close();
    process.exit(0);
  };
  process.on("SIGINT", shutdown);
  process.on("SIGTERM", shutdown);
}

if (process.argv[2] === "gateway") {
  await runGateway();
} else {
  await runRepl(); // 原有 REPL 逻辑抽成 runRepl()
}
```
（把现有 main.ts 的 REPL 主体包进 `async function runRepl()`，gateway 分支调用上面的 runGateway；imports 补齐 loadWecomConfig / WecomChannel / buildGateway。）

`apps/cli/package.json` dependencies 追加：
```json
    "@hermes-ts/gateway": "workspace:*",
    "@hermes-ts/channel-wecom": "workspace:*",
```
`pnpm install`。

- [ ] **Step 3: README 追加 gateway 节**

`README.md` 追加：
```markdown
## 企业微信 gateway (M3)
需要企业微信自建应用（管理后台 → 应用管理 → 自建 → 创建）。
环境变量（秘钥）：
    export WECOM_CORP_ID=...          # 企业 ID
    export WECOM_CORP_SECRET=...      # 应用 Secret
    export WECOM_AGENT_ID=...         # 应用 AgentId
    export WECOM_CALLBACK_TOKEN=...   # 接收消息 Token
    export WECOM_ENCODING_AES_KEY=... # EncodingAESKey（43 字符）
    export OPENAI_API_KEY=...         # 或 ANTHROPIC_API_KEY，与 REPL 相同
config.yaml：
    gateway:
      channel: wecom
      callbackPort: 8645
      callbackPath: /wecom/callback
启动：pnpm --filter @hermes-ts/cli start -- gateway
把「接收消息服务器URL」配成该回调地址的公网映射（自建应用需公网 HTTPS + 可信IP）。
```

- [ ] **Step 4: 验证 + 无 socket 组装测试通过 + 提交**

Run: `pnpm typecheck && pnpm lint && pnpm test` → PASS。
Run: `cd /Users/lirichen/Work/GithubRepo/hermes-ts && env -u WECOM_CORP_ID -u OPENAI_API_KEY pnpm --filter @hermes-ts/cli start -- gateway 2>&1 | head -3`
Expected: 因缺 OPENAI/WECOM 环境变量打印人类可读错误并退出（createProvider 或 loadWecomConfig 抛错，被 runGateway 的错误路径捕获或直接冒泡——确保有可读消息，无裸堆栈；若冒泡为裸堆栈，在 runGateway 外层包 try/catch 打印 message 并 exitCode=1）。

```bash
git add apps/cli README.md pnpm-lock.yaml && git commit -m "feat(cli): gateway subcommand wiring WeCom channel into Router"
```

---

## Task 12: M3 收尾 — 真实冒烟清单 + tag（owner 执行）

**Files:** 无代码改动（人工验收）

- [ ] **Step 1: 真实冒烟（需要企业微信自建应用 + 公网回调 + LLM key，owner 执行）**

前置：企业微信管理后台自建应用已配「接收消息服务器URL/Token/EncodingAESKey」，回调地址公网可达（可信IP 已加），有 OPENAI/ANTHROPIC key。
```bash
cd /Users/lirichen/Work/GithubRepo/hermes-ts
export WECOM_CORP_ID=... WECOM_CORP_SECRET=... WECOM_AGENT_ID=... WECOM_CALLBACK_TOKEN=... WECOM_ENCODING_AES_KEY=...
export OPENAI_API_KEY=...   # 或 anthropic
# config.yaml 里 gateway.channel: wecom
pnpm --filter @hermes-ts/cli start -- gateway
```

M3 完成标志（spec §7「手机上和 agent 对话」）：
1. 管理后台保存回调 URL 时，GET echostr 验证通过（后台显示配置成功）
2. 手机企业微信里给应用发一条消息 → 秒收「收到，处理中…」→ 随后收到 agent 的最终答案（**即时回执 + 最终答案** UX）
3. 发「列出当前目录文件」→ agent 调 shell 工具并把结果作为最终答案推回
4. 连续两条消息 → 第二条的回答能引用第一条上下文（会话持久化 + 串行化生效，不交错）
5. 重复回调（企业微信重试）不产生重复回答（MsgId dedup 生效）

- [ ] **Step 2: 打 tag 并推送**

```bash
git tag v0.3.0-m3 && git push && git push --tags
```

---

## Self-Review 记录

- **Spec 覆盖**：§7 gateway 三职责（托管适配器=WecomChannel/Task9、路由=Router/Task8、投递=SessionManager+send）；adapter 与"渲染"分离（WeCom 无编辑，渲染退化为 chunkText 纯函数/Task5）；会话并发=SessionManager per-key 串行化（Task7，关闭 M2 backlog 应用层半边）；CLI 不经 gateway 仍直连 kernel（REPL 分支保留）。§8 存储并发 idx 唯一约束（Task1，关闭 M2 backlog schema 半边）。
- **占位符扫描**：无 TBD；Task9 handlePost 里 conversationId 占位表达式已在正文标注为笔误并给出修正指令。
- **类型一致性**：`ChannelPort`/`InboundMessage`/`OutboundMessage` 在 kernel/gateway/channel-wecom 一致（Task2 定义，Task8/9 消费）；`WecomConfig`/`WecomRuntimeConfig` 形状在 Task9/10 对齐（实施 Task10 时确保字段名与 channel-wecom 的 WecomConfig 完全一致，若不一致以 channel-wecom 的 WecomConfig 为准）；`FetchLike` 在 token/message/adapter 一致（Task5 扩展 init 参数后全链路统一）。
- **已知妥协（进 M3 backlog）**：sessionKey→sessionId 内存映射跨进程重启丢失（会新建会话，历史仍在库但换新会话）；WeCom markdown 富文本未用（v1 纯文本）；图片/语音/文件/事件菜单未处理；智能机器人（WebSocket）模式未做；被动同步 XML 回复未实现（只用主动推送）；可信IP/HTTPS 由部署侧保障，代码不校验来源 IP。
- **企业微信协议以旧 Python `wecom_crypto.py` 为准**：PKCS7 块 32、key=base64(aeskey+"=")、iv=key[:16]、明文=16随机+4字节网络序长度+xml+receiveid、签名=SHA1(sorted 拼接)、token 刷新 errcode 40001/42001、dedup TTL 300s——均已落测试。
