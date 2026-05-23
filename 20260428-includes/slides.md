---
theme: default
title: includesを根本から理解する
highlighter: shiki
transition: slide-left
mdc: true
lineNumbers: true
---

# ActiveRecord の includes は<br>裏で何をやっているんだろう

<div class="pt-8 text-gray-500">
2026/04/28 gotanda.rb
<br>
@shilo_113
</div>

---

# 自己紹介

- Web系エンジニア（バックエンドやインフラを担当）
- 趣味は旅行✈️と韓国ドラマ🇰🇷
  - 再来月に初めてシンガポールに行きます。おすすめあれば教えてください。
- Ruby Kaigi 2026 楽しかったですね！
  - Day0：見事に飛行機が欠航したけど、意図せず生まれて初めての青函トンネル通過
  - Day1：
  - Day2：
  - Day3：
  - 去年に引き続き、2回目の参加
  - PicoRuby のワークショップとても楽しかったです！
  - 来年こそは Day4 のゴルフに参加したい
  - 行きは見事に飛行機が欠航になってしまった組です。

---
layout: center
class: text-center
---

# 本題

---

# みなさん、`includes`を使ってます？


```ruby
Post.includes(:comments).each do |post|
  post.comments.each { |c| puts c.body }
end
```
<v-click>

<br>

### こんなコード見る機会も多いはず

</v-click>


---

# なぜ includes を使用するのか？


```ruby
Post.includes(:comments).each do |post|
  post.comments.each { |c| puts c.body }
end
```

<v-click>

とりあえず書いとけば、
- N+1を防げるから
- パフォーマンスが良くなるから

</v-click>

<v-click>

<br>

### めっちゃわかる。

</v-click>


<v-click>

<br>

### でも、中で**何が起きているか**知っていますか？

</v-click>

---

# N+1問題のおさらい

```ruby
posts = Post.all
posts.each do |post|
  post.comments.each { |c| puts c.body }  # post数だけSQLが走る
end
```

```sql
SELECT * FROM posts;
SELECT * FROM comments WHERE post_id = 1;
SELECT * FROM comments WHERE post_id = 2;
SELECT * FROM comments WHERE post_id = 3;
-- ... N回繰り返す
```

→ `includes` を使うと解消できる。でも**どうやって？**

---
layout: two-cols
layoutClass: gap-8
---

# `preload`

関連を**別クエリ**で取得

```sql
SELECT * FROM posts;

SELECT * FROM comments
  WHERE post_id IN (1, 2, 3);
```

- JOINしない
- クエリ2本
- 結果セットが膨張しない

::right::

# `eager_load`

**LEFT OUTER JOIN** で一括取得

```sql
SELECT posts.*, comments.*
FROM posts
LEFT OUTER JOIN comments
  ON comments.post_id = posts.id;
```

- JOINする
- クエリ1本
- `where`で関連テーブルを絞れる

---
layout: center
class: text-center
---

# では`includes`はどちらを使う？

<v-click>

<br>

## 「**両方**使える」

</v-click>

<v-click>

<br>

### そして、どちらを使うかは**実行時に決まる**

</v-click>

---

# `includes`の正体

```ruby
Post.includes(:comments)
```

<v-click>

**これはただのフラグを立てているだけ**

```ruby
# ActiveRecord内部のイメージ
def includes(*args)
  self.includes_values += args  # フラグを積むだけ
  self
end
```

</v-click>

<v-click>

実際の戦略は **SQL生成時** に `eager_loading?` メソッドが決定する

</v-click>

---

# `eager_loading?` が分岐を決める

```ruby
def eager_loading?
  eager_load_values.any? ||
    (includes_values.any? && references_eager_loaded_tables?)
end
```

<br>

| 結果 | 動作 |
|---|---|
| `true` | `eager_load` 相当（LEFT OUTER JOIN） |
| `false` | `preload` 相当（クエリ2本） |

---

# `eager_load` が選ばれる3条件

**① `joins` を併用している**

```ruby
Post.includes(:comments).joins(:comments)
```

**② `where` のハッシュ条件で関連テーブルを参照**
　
```ruby
Post.includes(:comments).where(comments: { status: :published })
```

**③ `references` を明示している**
```ruby
Post.includes(:comments)
    .where("comments.status = 'published'")
    .references(:comments)
```

---
layout: center
class: text-center
---

# ここが落とし穴

---

# 意図せずJOINになっている

```ruby
# 「N+1対策でincludesを書いた」つもりが...
Post.includes(:comments).where(comments: { status: :published })
```

<v-click>

```sql
-- 実はLEFT OUTER JOINが走っている
SELECT posts.*, comments.*
FROM posts
LEFT OUTER JOIN comments ON comments.post_id = posts.id
WHERE comments.status = 'published'
```

</v-click>

<v-click>

`has_many` の場合、結果セットが **post数 × comment数** に膨張する

→ 気づかないうちにパフォーマンスが変わっている

</v-click>

---

# 実践的な使い分け

| 関連の種類 | 推奨 | 理由 |
|---|---|---|
| `belongs_to` / `has_one` | `eager_load` が選択肢に入る | 1対1なので結果セット膨張が小さい |
| `has_many` | `preload` | JOINで行数が爆発しやすい |

<v-click>

**`has_many` + 絞り込みが必要な場合 → 2段階クエリ**

```ruby
# 「publishedなcommentを持つpost」に絞り込みたい場合
post_ids = Comment.where(status: :published).select(:post_id)
posts    = Post.where(id: post_ids).preload(:comments)
# ⚠️ preloadは全commentを読み込む点に注意
#    関連レコード自体を絞り込みたい場合は別途スコープが必要
```

</v-click>

---

# まとめ

- `includes` は**ただのフラグ**。SQL生成時に `eager_loading?` が分岐を決める
- `where(association: { ... })` と書くだけで**意図せずJOIN**になる
- `has_many` に `eager_load` を使うと**結果セットが膨張**する
- 関連の種類を意識して `preload` / `eager_load` を**明示的に選ぶ**場面を作ろう

<v-click>

<br>

### `includes` は便利だけど、中身を知って使おう

</v-click>

---
layout: center
class: text-center
---

# ありがとうございました

<div class="pt-4 text-gray-500">
余談ですが、今回初めて slidev 使用しました。めっちゃ良い。
</div>
