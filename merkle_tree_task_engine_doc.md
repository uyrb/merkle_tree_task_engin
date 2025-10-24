# 🧩 Merkle_tree Task Engine
### Rubyによる自己再帰タスクツリー設計思想と実装構造

---

## 📘 概要

**Merkle_tree Task Engine** は、
Ruby のブロックと再帰構造を用いて設計された「自己再帰・自己書き換え型タスクツリーシステム」です。  

このシステムは、**「タスクの初期化」と「タスクの実行」を分離せず統一的に扱う** ことを目的としており、
ノード（task）が自身の `func` を動的に書き換えながら進化していくことで、
複雑なタスク制御・ツリー駆動ロジックを最小構文で表現できます。

> 💡 言い換えると：
> `initialize` フェーズと `update` フェーズを “一つの再帰構造” に融合したタスクエンジン。

---

## 🧠 設計思想

### 空間A → 空間B の構造
```ruby
o.Task :__task_name do |o|
  o.Code do
    # ここがタスクの本体ロジック
  end
end

---

空間A (Taskブロック)：
ノードを生成する初期化フェーズ

空間B (Codeブロック)：
実行ループとして繰り返し呼ばれる処理本体

b.func = b.func[b].func という1行により、
空間Aで生成されたブロック（初期状態）が、空間Bへと自己上書きされていきます。
結果として、「初期化処理 → 実行処理」の切り替えをコード上で明示せずに統一できます。

---
基本構造 tree_task.rb
module Merkle_tree_m
  attr_accessor :sym, :up, :func, :task

  def initialize(sym: nil, up: nil, &proc)
    @sym  = sym
    @up   = up
    @func = proc
    @task = []
  end

  def _taskloop
    task.each { |b| b.func = b.func[b].func }
  end

  def Main(sym, &proc)
    Task(sym, &proc)
    nil while not _taskloop.empty?
  end

  def Task(sym = :task, &proc)
    task << self.class.new(sym: sym, up: self, &proc)
  end

  def Code
    self.class.new do
      yield
      _taskloop
      self
    end
  end
end

拡張モジュール　tree_task_ex.rb

module Merkle_tree_m_ex
  def TOP_NODE
    return up.up.TOP_NODE if up.up
    self
  rescue
    self
  end

  def TOP_SYM
    self.TOP_NODE.sym
  end

  def delete
    up.task.delete self
    task.clear
  end

  def task_delete
    delete
    up.delete
  end

  def search_up(sym)
    return self if self.sym =~ /#{sym}/
    return up.search_up(sym) if up
    false
  end

  def search_up_all_sym = up ? [up.search_up_all_sym, sym] : []

  def search_down_test(o = self)
    o.task.each.inject([]) do |stack, m|
      unless m.task.empty?
        next [stack, m.search_down_test(m)]
      end
      [stack, m.sym]
    end << o.sym
  end

  # --- lazy task ---
  def delete_lazy(wait)
    Task do |o| o.Code do
      if wait < 0
        yield o if iterator?
        delete
      else
        wait -= 1
      end
    end end
  end

  def create_lazy(wait)
    Task do |o| o.Code do
      (wait < 0) ? yield(o) : (wait -= 1)
    end end
  end

  # --- tree visualization ---
  def task_list_p(indent = 1, first_call = true)
    indent_str = '  ' * indent
    if first_call
      p "-#{self.sym} (up:#{self.TOP_SYM})"
    end
    unless task.empty?
      task.each_with_index do |o, i|
        p "#{indent_str}├─ #{o.sym} (up:#{o.up ? o.up.sym : 'none'})"
        o.task_list_p(indent + 1, false) unless o.task.empty?
      end
    end
    p "-----------"
  end
end

---
Merkle_treeクラス (Merkle_tree.rb)
class Merkle_tree
  include Merkle_tree_m
  include Merkle_tree_m_ex
  attr_accessor :Flandoll, :Scarlet

  def initialize(**_)
    super
    @Flandoll = []
    @Scarlet  = Hash.new { |h, k| h[k] = [] }
  end
end

---
実行イメージ
Merkle_tree.new.Main :top_node do |o|
  o.Code do
    if o.task.empty?
      o.Flandoll << :menu
    end

    -> &b do
      lambda do |f|
        lambda{|x|lambda{|y| f[x[x]] [y]}}[
          lambda{|x|lambda{|y| f[x[x]] [y]}}]
      end[ lambda{|f|lambda{|n| b[n , &f] }}]
    end.yield do |(o, rb), &f|
      case o.Flandoll.shift
      when nil
        true
      when -> rb { Merkle_tree.loading(o, rb) }
      when -> rb {
          o.Task :"__#{rb}_task" do |o|
            o.Code do
              o.Main :"#{rb}" do |o|
                Merkle_scene.new o, rb
                o.Code { f[[o, rb]] }
              end
            end
          end
        }
      end
    end[ [o, nil] ]
  end
end

---
タスク生成と実行の流れ（mermaid図）
flowchart TD

  subgraph A["空間A : Task（初期化フェーズ）"]
    T1["o.Task :__example_task"]
    C1["=> 子ノード生成（sym, func, up, task）"]
  end

  subgraph B["空間B : Code（実行フェーズ）"]
    T2["o.Code do ... end"]
    C2["yieldブロック内で処理本体"]
    C3["_taskloop により func 再代入"]
    C4["b.func = b.func[b].func"]
  end

  subgraph L["ループ制御 : _taskloop"]
    L1["task.each"]
    L2["func再帰呼び出し"]
    L3["funcが進化（自己書き換え）"]
  end

  T1 -->|初期化| T2
  T2 -->|yield| C2
  C2 -->|再帰呼び出し| C3
  C3 -->|代入| C4
  C4 -->|次フレーム| L1
  L1 --> L2
  L2 --> L3
  L3 -->|戻る| T2

---
ツリー構造の視覚モデル
graph TD

  M["Main(:top_node)"]
  A["Task(:menu_task)"]
  B["Code{menuロジック}"]
  C["Task(:sub_task_1)"]
  D["Code{sub処理}"]
  E["Task(:sub_task_2)"]

  M --> A
  A --> B
  B --> C
  B --> E
  C --> D

---
func の自己書き換えイメージ
sequenceDiagram
  participant Node as Taskノード
  participant Func as func(状態)
  participant Loop as _taskloop

  Node->>Func: 初期状態(func0)
  Func->>Node: Codeブロック内yield実行
  Loop->>Func: func = func[Node].func に置換
  Func->>Func: 新しいfunc1に変化
  Loop-->>Node: 次サイクルでfunc1実行

---
lazy taskのライフサイクル図
sequenceDiagram
  participant Parent as 親ノード
  participant Lazy as create_lazy(wait)
  participant Loop as _taskloop

  Parent->>Lazy: waitフレーム後に子ノード作成
  Loop->>Lazy: wait -= 1
  Lazy-->>Loop: wait < 0 ? yield(o) : 継続
  Lazy->>Parent: 子タスク生成完了
  Loop->>Parent: 通常のCodeロジックに戻る
---
概念まとめ図
flowchart LR
  subgraph Phase["1. 構築フェーズ"]
    A1["Main"]
    A2["Task"]
    A3["Code"]
  end

  subgraph Phase2["2. 実行フェーズ"]
    B1["_taskloop"]
    B2["func自己代入"]
    B3["再帰呼び出し"]
  end

  A1 --> A2 --> A3
  A3 --> B1
  B1 --> B2 --> B3 --> A3

  特徴と利点
  項目	内容
  再帰的タスク駆動	各ノードが自分自身の処理を更新しながら再帰実行
  状態の内包化	func が状態を保持するため、外部ステート不要
  柔軟な構造	任意のツリー構造でタスク依存関係を表現可能
  高い拡張性	lazy実行、削除、探索などのメソッドで制御自在
  軽量・依存なし	標準Rubyのみで動作。外部ライブラリ不要

応用例

  非同期処理・ステートマシン・コルーチン的スケジューラ

  ゲームエンジンのシーングラフ

  AI行動ツリー

  自己再帰構造のDSL基盤

総評

  「タスクをオブジェクトで管理する」ではなく、
  「関数そのものが進化しながらツリーを形成する」 という発想。

  Merkle_tree Task Engine は、
  Rubyの柔軟なProcモデルと再帰思考を融合させた、
  状態進化型タスクシステムの一つの完成形 です。

ライセンス

  MIT（予定）

  🏷️ 作者メモ

  この構造は試行錯誤の末に設計されたものです。
  もしこの考え方があなたの開発や思考に刺激を与えたなら、ぜひこの思想をベースに
  新しいタスクエンジン・アーキテクチャを考案してみてください。

---

__END__

# 6. 設計上の利点・課題

[→ 柔軟性・再帰的拡張性へ移動](#柔軟性・再帰的拡張性)

## 柔軟性・再帰的拡張性
ここに本文を書きます。
