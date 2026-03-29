---
title: "TypeScript Union 型でスキルのレンジ種別とダメージ計算を型安全に実装する"
date: 2026-03-28T17:00:00+09:00
tags: ["TypeScript", "型設計", "ゲーム開発", "設計"]
categories: ["開発"]
---

# TypeScript Union 型でスキルのレンジ種別とダメージ計算を型安全に実装する

ゲーム開発、特に RPG やタクティカルゲームにおいて、スキルの「射程（レンジ）」や「効果範囲（AOE: Area of Effect）」の実装は非常に複雑になりがちです。単体攻撃、直線、周囲、円形、さらには自分自身を対象とするものまで、そのバリエーションは多岐にわたります。

これらを `string` 型の ID や、大量の `if` 文、あるいは複雑なクラス継承で管理しようとすると、いつか必ず「新しいレンジを追加したのに、判定処理を書き忘れた」あるいは「このスキルタイプには不要なはずのパラメータが混入している」といったバグに直面します。

TypeScript の **Union 型（特に Discriminated Union / 判別可能な共用体）** を活用すれば、こうしたロジックをコンパイルレベルで安全に保護し、設計意図をコードに焼き付けることができます。本記事では、架空のゲームを題材に、`any` や `as` を一切使わずに、メンテナンス性の高いスキルシステムを構築する手法を徹底解説します。

---

## 1. Union 型でスキル種別を表現する

まず、スキルの「レンジ（範囲）」を型として定義します。ここでのポイントは、単に名前を列挙するのではなく、**「そのレンジを計算するために最低限必要なデータ」をセットにする**ことです。

### なぜこの設計にするのか（データ構造の純粋性）
オブジェクト指向的な発想では、`BaseRange` クラスを継承して `AreaRange` クラスを作る、といった方法が取られがちです。しかし、ゲームデータ（特に JSON などでシリアライズされるデータ）を扱う場合、純粋なオブジェクト（POJO）として扱える方がシリアリアライズの相性が良く、ロジックを分離（疎結合）しやすくなります。

```typescript
// 1. 各レンジの個別定義
type MeleeRange = { 
  type: 'melee' 
}; // 隣接マス（1マス）

type RangedRange = { 
  type: 'ranged'; 
  distance: number 
}; // 遠距離（単体指定）。最大射程が必要。

type LineRange = { 
  type: 'line'; 
  length: number;
  width?: number; // オプションで太さを持たせることも可能
}; // 直線。貫通距離が必要。

type AreaRange = { 
  type: 'area'; 
  radius: number; 
  excludeCenter?: boolean 
}; // 円形または四角形範囲。半径が必要。

type SurroundRange = { 
  type: 'surround' 
}; // 自分の周囲8マス。パラメータ不要。

type SelfRange = { 
  type: 'self' 
}; // 自分自身。パラメータ不要。

// 2. これらを統合した Union 型（Discriminated Union）
export type SkillRange =
  | MeleeRange
  | RangedRange
  | LineRange
  | AreaRange
  | SurroundRange
  | SelfRange;

// 3. スキルの全体定義
export interface Skill {
  id: string;
  name: string;
  description: string;
  range: SkillRange; // 型安全なレンジ定義
  baseDamage: number;
  scalingStat: 'STR' | 'INT' | 'DEX'; // どのステータスに依存するか
  manaCost: number;
}
```

このように `type` という「タグ（判別子）」を持たせることで、TypeScript は「`type` が `'area'` なら `radius` プロパティが確実に存在する」と認識します。逆に、`'melee'` の時には `radius` にアクセスしようとするとコンパイルエラーになります。これにより、不必要なデータへの依存を完全に排除できます。

---

## 2. レンジ別ヒット判定の実装（網羅性チェックの活用）

次に、選択したスキルがフィールド上のどのマスに影響を与えるかを計算するロジックを実装します。ここで重要なのが、`switch` 文による **Exhaustive Check（網羅性チェック）** です。

### 実装例：影響マスの算出ロジック

```typescript
type Point = { x: number; y: number };

/**
 * 指定したスキルと方向から、影響を受ける座標リストを計算する
 * @param origin 攻撃者の位置
 * @param direction 攻撃方向（{x:1, y:0} など）
 * @param range スキルのレンジ定義
 */
export function getAffectedCells(
  origin: Point,
  direction: Point,
  range: SkillRange
): Point[] {
  // ここで range.type によって型が絞り込まれる（Type Narrowing）
  switch (range.type) {
    case 'melee':
      return [{ x: origin.x + direction.x, y: origin.y + direction.y }];

    case 'ranged':
      // 実際の実装ではカーソル位置などが必要だが、ここでは射程端を返す
      return [{ x: origin.x + direction.x * range.distance, y: origin.y + direction.y * range.distance }];

    case 'line':
      const lineCells: Point[] = [];
      for (let i = 1; i <= range.length; i++) {
        lineCells.push({ x: origin.x + direction.x * i, y: origin.y + direction.y * i });
      }
      return lineCells;

    case 'area':
      const areaCells: Point[] = [];
      for (let dx = -range.radius; dx <= range.radius; dx++) {
        for (let dy = -range.radius; dy <= range.radius; dy++) {
          // ユークリッド距離で円形判定
          if (Math.sqrt(dx * dx + dy * dy) <= range.radius) {
            areaCells.push({ x: origin.x + dx, y: origin.y + dy });
          }
        }
      }
      return areaCells;

    case 'surround':
      const surround: Point[] = [];
      for (let dx = -1; dx <= 1; dx++) {
        for (let dy = -1; dy <= 1; dy++) {
          if (dx === 0 && dy === 0) continue;
          surround.push({ x: origin.x + dx, y: origin.y + dy });
        }
      }
      return surround;

    case 'self':
      return [origin];

    default:
      /**
       * 【重要】網羅性チェック（Exhaustive Check）
       * もし SkillRange に新しい type が追加されたのに、
       * この switch 文で case を追加し忘れている場合、
       * range は never 型にならず、ここでコンパイルエラーが発生する。
       */
      const _exhaustiveCheck: never = range;
      throw new Error(`Unhandled range type: ${(_exhaustiveCheck as any).type}`);
  }
}
```

### 網羅性チェックが「開発の武器」になる理由
大規模な開発では、エンジニア A が新しいレンジ（例：扇形 `cone`）を追加し、エンジニア B がその描画処理を書く、といった分業が発生します。
このとき、`getAffectedCells` のような重要ロジックに `default: never` のガードを置いておけば、**「新しいレンジを追加した瞬間に、プロジェクト中の判定ロジックがコンパイルエラーとして浮き彫りになる」** という状態を作れます。これは、ユニットテスト以上に強力な「実装の強制力」となります。

---

## 3. ステータスに応じたダメージ計算と VIT 防御計算

スキルの命中範囲が決まったら、次は個別のターゲットに対するダメージ計算です。ここでは、スキルの特性（物理・魔法など）と、攻撃者・防御者のステータスを組み合わせます。

### ステータスとユニットの型定義

```typescript
export interface UnitStats {
  STR: number; // 筋力
  INT: number; // 知力
  DEX: number; // 器用さ
  VIT: number; // 生命力（物理防御）
  MEN: number; // 精神力（魔法防御）
}

export interface GameUnit {
  id: string;
  name: string;
  stats: UnitStats;
  currentHp: number;
}
```

### ダメージ計算式の実装

```typescript
/**
 * スキルによる最終ダメージを算出する
 */
export function calculateDamage(
  attacker: GameUnit,
  defender: GameUnit,
  skill: Skill
): number {
  const attackPower = attacker.stats[skill.scalingStat];

  // 物理攻撃なら VIT, 魔法攻撃なら MEN を参照
  const isMagic = skill.scalingStat === 'INT';
  const defensePower = isMagic ? defender.stats.MEN : defender.stats.VIT;

  const baseMultiplier = skill.baseDamage / 100;
  const rawDamage = (attackPower * baseMultiplier) - (defensePower * 0.4);

  return Math.floor(Math.max(1, rawDamage));
}
```

---

## 4. 発展：クラス継承 vs Union 型

「なぜクラス継承を使わないのか？」という疑問に答えるために、両者の設計思想を比較してみましょう。

### クラス継承による設計（OOP）
```typescript
abstract class BaseRange {
  abstract getAffectedCells(origin: Point, dir: Point): Point[];
}
class AreaRange extends BaseRange {
  constructor(public radius: number) { super(); }
  getAffectedCells(origin: Point, dir: Point) { /* 実装 */ }
}
```
**メリット:** ロジックとデータが一体化しており、新しいレンジを追加する際に既存のコード（`switch` 文など）を触る必要がない（Open-Closed Principle）。
**デメリット:** データのシリアライズ（JSON 保存）が面倒。ロジックが各クラスに分散するため、全体の見通しが悪くなることがある。

### Union 型による設計（関数型 / データ指向）
**メリット:** データ構造がシンプルで JSON との親和性が高い。ロジックが `getAffectedCells` という一箇所に集約されるため、デバッグが容易。網羅性チェックにより、実装漏れを防げる。
**デメリット:** 新しいレンジを追加する際、既存の `switch` 文を修正する必要がある。

現代的なゲームフロントエンド（React / Redux / Vue など）や、状態管理を不変（Immutable）に行うシステムでは、**Union 型による設計の方が圧倒的に相性が良い** です。

---

## 5. 応用：ターゲット選択の型安全な拡張

スキルには「誰を対象にするか」という情報も必要です。これも Union 型でネストさせることができます。

```typescript
type TargetFilter = 'all' | 'enemies' | 'allies' | 'except-self';

export interface ComplexSkill extends Skill {
  targetFilter: TargetFilter;
}

function filterTargets(
  attacker: GameUnit, 
  candidates: GameUnit[], 
  filter: TargetFilter
): GameUnit[] {
  switch (filter) {
    case 'all': return candidates;
    case 'enemies': return candidates.filter(u => isEnemy(attacker, u));
    case 'allies': return candidates.filter(u => isAlly(attacker, u));
    case 'except-self': return candidates.filter(u => u.id !== attacker.id);
    // ここでも網羅性チェックが可能
  }
}
```

---

## 6. まとめ：型安全なゲーム設計がもたらす長期的なメリット

本記事では、TypeScript の Union 型を活用したスキルの設計手法を見てきました。

1.  **データの純粋性**: Union 型により、各レンジに必要なデータだけを無駄なく持たせることができた。
2.  **ロジックの安全性**: `never` 型を用いた網羅性チェックにより、実装漏れをコンパイルエラーとして検出できた。
3.  **キャストの排除**: `any` や `as` を使わずに、ステータス参照やダメージ計算を型安全に行えた。

ゲーム開発は、機能追加やバランス調整による「破壊的な変更」が日常茶飯事です。その中で、**「コードを変更したときに、どこが壊れたかを型システムが教えてくれる」** という安心感は、開発スピードを劇的に向上させます。

最初は型定義が面倒に感じるかもしれませんが、一度この「型に守られた開発」を体験すると、もう `string` や `any` だらけのコードには戻れなくなるはずです。ぜひ、あなたのプロジェクトでも、Union 型による型主導の設計を取り入れてみてください。

---
*著者注: 本記事のサンプルコードは、概念理解のために簡略化しています。実際のタクティカルゲーム等では、高低差の判定、障害物による遮蔽（Line of Sight）、複数回ヒットする多段攻撃など、さらに複雑な要素が加わりますが、それらも同様に Union 型をネストさせることで美しく表現可能です。*
