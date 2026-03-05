---
title: "テスト前提で設計したWebアプリのハンズオン - 読書管理アプリ その6"
date: 2026-03-05T23:00:00+09:00
tags: ["設計", "DDD", "クリーンアーキテクチャ", "TypeScript", "振り返り"]
categories: ["開発"]
---

## はじめに

Part5でUIまで実装が終わった。機能として動くものはできた。テストも15件、全パス。

ここで一度立ち止まって、**なぜこの設計になったのか**を振り返る。

正直に言う。最初の要件はこうだった。

> 「クリーンアーキテクチャっぽく、テスタブルにしたい」

それだけだった。クリーンアーキテクチャを理解して設計したわけじゃない。
**結果的にクリーンアーキテクチャを遂行したのは生成AIだ。**

そしてPart3以降、自分はほとんど手を動かさなくなった。
設計ドキュメントをAIに渡したら、実装サイクルがほぼ自動で回るようになったからだ。

途中でこう思った。

**「俺いる必要なくね？」**

この記事はその問いに向き合いながら、設計を改めて言語化したものだ。

---

## このシリーズ自体がTDDだった

書き終えてから気づいたことがある。

このシリーズのサイクルはこうだった。

```
AIに実装させる
  ↓
動かす・読む・会話する（認識のズレを検出）
  ↓
ズレを言語化してAIにフィードバック
  ↓
納得したら記事にする
```

ソフトウェアのTDDは「Red → Green → Refactor」だけど、
自分がやっていたのはこれだ。

- **Red** → 認識がズレていると感じる
- **Green** → 会話して納得する
- **Refactor** → 記事として言語化する

**人間がテストケースになっていた。**

TDDが「仕様をテストで表現する」なら、自分がやっていたのは「理解をフィードバックループで検証する」だ。構造は同じだと思う。

---

## 誤っていた認識たち

振り返ると、理解がズレていた箇所がいくつかあった。

### 「PolicyはVOと1-1になる」と思っていた

最初、`ReadingStatusPolicy`が`ReadingStatus`に対応しているのを見て、PolicyはVOと対になるものだと思っていた。

違う。今回たまたま1-1になっているだけだ。

Policyの本質は**複数のEntityやVOをまたいだ条件判定**だ。たとえば「同じ本に同じユーザーが2回レビューできない」というルールは、Book・User・Review[]をまたぐ。これをPolicyに出す。

VOに閉じるルールならVOに書けばいい。Entityをまたぐ判定が必要になったときにPolicyの出番だ。

### 「Domainは外部を知らない」という捉え方が逆だった

「DomainはDBを知らない」という言い方をよくするが、正確には逆だ。

**外部がDomainを知っている。**

向きの問題だ。Domainが何かを避けているのではなく、依存の矢印がすべてDomainに向かって刺さっている。

```
Route Handler
  ↓
Service
  ↓
Repository interface
  ↓
Domain（Entity / ValueObject / Policy）
```

この向きがわかって初めて、Serviceの役割も見えた。

### Serviceが何をするのかわかっていなかった

依存の向きが腑に落ちるまで、Serviceが何者なのかずっと曖昧だった。

答えはシンプルだった。**フローだけ持つ接着剤。**

```typescript
async startReading(id: string): Promise<Book> {
  const book = await this.repo.findById(id);   // 取得
  if (!book) throw new NotFoundError(...);
  book.changeStatus(ReadingStatus.Reading);     // Entityのルールに従う
  await this.repo.save(book);                  // 保存
  return book;
}
```

ServiceはDomainのルールを**自分で判定しない**。
`canTransition`を自分で呼ばない。`book.changeStatus()`に委ねるだけだ。
判定はEntity・VO・Policyが持っている。Serviceはその結果を使ってフローを組む。

依存の向きがわかって、はじめてServiceの「フローだけ持つ」という役割が見えた。この順番で理解する必要があった。

---

## 設計の全体像

改めて整理する。

### ValueObject（VO）

ルール付きの値。型の拡張。

```typescript
export class ISBN {
  constructor(public readonly value: string) {
    if (!value.match(/^\d{13}$/)) {
      throw new Error("ISBNは13桁の数字");
    }
  }
}
```

`ISBN`型のインスタンスが存在する時点で、13桁の数字であることが保証される。

### Policy

条件判定だけ。副作用なし、`boolean`を返すだけ。

```typescript
static canTransition(from: ReadingStatus, to: ReadingStatus): boolean {
  const rules = {
    [ReadingStatus.Unread]: [ReadingStatus.Reading],
    [ReadingStatus.Reading]: [ReadingStatus.Completed],
    [ReadingStatus.Completed]: [ReadingStatus.Reading],
  };
  return rules[from].includes(to);
}
```

### Entity

複数のVOが絡むルールを持つ構造体。Policyを呼んで状態遷移を制御する。

```typescript
changeStatus(to: ReadingStatus): void {
  if (!ReadingStatusPolicy.canTransition(this.status, to)) {
    throw new Error(`${this.status}から${to}への変更は不可`);
  }
  this.status = to;
}
```

### Service

ユースケースのフロー。ルール判定はDomainに委ねる。

### Repository

DBとの橋渡し。Domainはこの存在を知らない。

---

## なぜテストが書きやすかったか

DomainはDBもHTTPも知らないので、インスタンスを作るだけでテストできる。

```typescript
test("積読から直接読了はエラー", async () => {
  const service = new BookShelfService(createMockRepo());
  await service.addBook("1", "Clean Code", "9784048860000");

  await expect(service.completeReading("1")).rejects.toThrow(
    "UnreadからCompletedへの変更は不可",
  );
});
```

MockはMapベースのインメモリ実装をテストファイル内に書いただけ。
Prismaもサーバーも起動していない。

依存の向きを守った結果としてテストが書きやすくなった。
テストのために設計したのではなく、設計が正しいからテストが書けた。

---

## クリーンアーキテクチャの簡略版として

この設計はクリーンアーキテクチャの考え方をベースにしている。

参考: [The Clean Architecture – Clean Coder Blog](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

フルだと4層あって、PresenterやViewModelも分離する。
今回省略しているのはPresenterとUseCase interfaceの分離。
規模に対して過剰になる部分は省いた。

**依存の向きだけは守った。それだけで十分にテスタブルな設計になった。**

ただしこれを最初から理解して設計したわけじゃない。
「クリーンアーキテクチャっぽく、テスタブルにしたい」と言っただけで、
構造を作ったのはAIだ。自分は後から理解した。

---

## 「俺いる必要なくね？」に対する答え

Part3以降で手を放したせいで、設計の理解が抜けていた。
PolicyとVOの関係も、依存の向きも、Serviceの役割も、改めて問われると曖昧だった。

実装が自動化されることと、設計を理解していることは別の話だ。

AIが得意なのは**仕様が決まった実装**だ。
「何を仕様にするか」と「その設計が正しいかどうか」は人間が判断する必要がある。
今回それをサボっていた。

そしてもう一つ。
「NDL連携を入れる」「Reviewを今入れない」という判断はAIがしていない。
何を作るかを決めたのは自分だ。AIは決まったことを実装した。

**「俺いる必要なくね？」の答えは、理解をサボったら本当にいらなくなる、だと思う。**

---

## まとめ

- 最初の要件は「クリーンアーキテクチャっぽく、テスタブルにしたい」だけだった
- 設計を遂行したのはAIで、自分は後から理解した
- PolicyはVOと1-1ではない。複数EntityをまたぐときにPolicyが出てくる
- 依存の向きは「Domainが外部を知らない」ではなく「外部がDomainを知っている」
- Serviceの役割はフローだけ。依存の向きがわかって初めて見えた
- このシリーズ自体がTDDだった。人間がテストケースになっていた
- 実装を自動化しても、設計の理解は自分でやる必要がある

次のPartではUser・ReviewのDB拡張とReviewServiceを実装する予定。
今度は手を動かす部分を意識的に残す。
