# ドキュメント（初版）

## 目次

1. 概要  
　・[「Merkle_tree式タスクツリーエンジン」とは何か](#merkle_tree式タスクツリーエンジンとは何か)  
　・[ゲーム基盤としての位置づけ](#ゲーム基盤としての位置づけ)

2. 設計思想  
　・[「空間A→空間B」構造(TaskとCodeの関係)](#空間a空間b構造taskとcodeの関係)  
　・[関数の自己再代入(b.func--b.funcbfunc)によるタスク進化](#関数の自己再代入によるタスク進化)  
　・[「initialize」を「update」で上書きする発想](#initializeをupdateで上書きする発想)

3. 実装構造  
　・[Merkle_tree_mモジュールの解剖](#merkle_treemモジュールの解剖)  
　・[Main--Task--Codeの流れ](#maintaskcodeの流れ)  
　・[Merkle_treeクラスによる拡張と状態保持](#merkle_treeクラスによる拡張と状態保持)  
　・[再帰ループによるノード実行フロー](#再帰ループによるノード実行フロー)

4. 実行例（最小構成サンプル）  
　・[Main:top_nodeの例と実際の再帰挙動の図解](#maintop_nodeの例と実際の再帰挙動の図解)  
　・[ステップごとの「funcの書き換え」可視化](#ステップごとのfuncの書き換え可視化)

5. 応用例  
　・[Merkle_tree_m_ex](#Merkle_tree_m_ex)  
　・[Lazy_delete](#lazy_delete)  
　・[Lazy_create](#lazy_create)  
　・[将来的なAIタスクツリー応用](#将来的なaiタスクツリー応用)

6. 設計上の利点・課題  
　・[柔軟性・再帰的拡張性](#柔軟性再帰的拡張性)  
　・[理解コスト・デバッグ性](#理解コストデバッグ性)  
　・[将来的な改善ポイント（例:視覚化・トレース機構）](#将来的な改善ポイント例視覚化トレース機構)

7. まとめ  
　・[「Merkle_tree式」の特徴](#merkle_tree式の特徴)  
　・[終わりに装との比較(ECS,Coroutine,TaskGraphなど)](#終わりに装との比較ecs-coroutine-taskgraphなど)  
　・[この構造がもたらす抽象化の自由度](#この構造がもたらす抽象化の自由度)

---

## 概要
### 「Merkle_tree式タスクツリーエンジン」とは何か

```ruby
  # --- tree_task.rb ---
  module Merkle_tree_m
    attr_accessor :sym , :up , :func , :task
    def initialize sym: nil , up: nil , &proc
      @sym      = sym
      @up       = up
      @func     = proc
      @task     = []
    end
    def _taskloop
      task.each do | b | b.func = b.func[b].func end
    end
    def Main sym, &proc
      Task sym , &proc
      nil while not _taskloop.empty?
    end
    def Task sym = :task, &proc
      task << self.class.new(sym:sym , up:self , &proc)
    end
    def Code
      self.class.new do
        yield
        _taskloop
        self
      end
    end
  end
```

使い方
```ruby
class Merkle_tree
  include Merkle_tree_m
end
Merkle_tree.new.Main :root do |o|
  p "hallo Merkle_tree"
  o.Code do
    exit
  end
end
```

### ゲーム基盤としての位置づけ
ゲームのタスクシステムをシンプルにスマートな構造で提供
複雑な言語機能なしに僅かなコードで実装します

Merkle_tree式タスクツリーエンジンは、Rubyで動的に再帰・自己再代入を行う構造的タスク管理システムです
ゲームやツールにおけるメインループや非同期的処理を、少量のコードで表現することを目的としています

---

## 設計思想
### 「空間A→空間B」構造(TaskとCodeの関係)

これは主に初期化と実装をできる限り近い位置に置く事を目指して作られました
よって
```ruby
Merkle_tree.new.Main :root do |o|
  # 初期化（空間A）
  n = 0
  o.Code do
    # 実装 メインループ （空間B）
    p n+=1
  end
end
```
これでゲームループが出来上がっています

### 関数の自己再代入(b.func = b.func[b].func)によるタスク進化

つまりこれは
### b.func = b.func[b].func
空間Aへ空間Bを代入するコードになっています
そして空間Bへ代入したあとの空間Bの戻り値は空間Bです
```ruby
def _taskloop
  task.each do | b | b.func = b.func[b].func end
end
```
ざっくりと説明するなら
空間Aの Task do ~ end は1度だけ実行され
空間Bの Code do ~ end はループして実行され続けます

「initialize」を「update」で上書きするという発想です

+----------------------------------+
|           Merkle_tree            |
|  (タスクツリーのルートノード)   |
+----------------------------------+
              |
              v
      +----------------+
      | Task (空間A)   | ← ノード生成（初期化）
      +----------------+
              |
              v
      +----------------+
      | Code (空間B)   | ← 実行フェーズ
      +----------------+
              |
              v
      +----------------+
      | _taskloop      | ← 再帰実行・funcの自己更新
      +----------------+

「Task」で作った空間Aが「Code」で上書きされ、
_taskloop によって再帰的に自分自身を進化させていく

---

## 実装構造
### Merkle_tree_mモジュールの解剖

タスクを削除するための最低限のサンプル
```ruby
module Merkle_tree_m_ex
  def delete
    up.task.delete self
    task.clear
  end
end
```
自身ノードを保有する上のノード(up)のタスクから自身を削除する
そして自身の持つタスクをクリアする

### Main / Task / Code の流れ

Main(:root)
   │
   └─▶ Task(:example_task)
          │
          └─▶ Code do
                ├─ 実行処理
                ├─ Task(:child1)
                └─ Task(:child2)
各 Task は自分の Code 内で子タスクを追加できる。
Main は再帰的に全タスクを呼び出し、実行フェーズを維持する


この三つの関数のみで
ノードの生成、追加、ループを提供します
ルールは２つ。
### Taskの後に必ずCodeを呼ぶ
### エントリーポイントはMain（応用は後述）
まずエントリーポイントでメインを呼びます
```ruby
Merkle_tree.new.Main :root do |o|
```
このMainの中にはTaskが内包されています
よって次は
```ruby
  o.Code do end
```
これが必要になる

そしてその中でTaskを作るには
```ruby
  o.Code do
    o.Task(:hello) { |t| t.Code { puts "Hello task run"; t } }
  end
```
ただしこれでは無限ループでTaskを追加し続けているので動かない

```ruby
o.Code do
  p "Code do"
  next unless o.task.empty?
  puts "[INIT] creating tasks under #{o.sym}"
  o.Task(:hello) { |t| t.Code { puts "Hello task run"; t } }
end
```
ここまで書く事によって初めて動作する
o.Task(:hello)は :root タスク下に生成されている
つまり
root.task[0] == ＜:halloタスク＞

まずはMainについては、単なるエントリーポイントとしてのおまじないとしてしておき
TaskとCodeの動作から理解していくのが好ましい

### Merkle_treeクラスによる拡張と状態保持
```ruby
Merkle_tree.new.Main :root do |o|
  n = 0
  o.Code do
    p "Code do"
    next unless o.task.empty?
    o.Task(:hello) { |t| t.Code { puts "Hello task run#{n+=1}"; t } }
  end
end
```
空間Aで宣言した n = 0 は
空間Bで生成したタスクのさらに子タスクのスコープを超えて状態保存される
これはRuby言語の特性である


### 再帰ループによるノード実行フロー
複数関数が相互に影響しあう見慣れない再帰構造のため、
極めて理解が難しいとは思われるが、Tree探索はここで行われる
```ruby
def _taskloop
  task.each do | b | b.func = b.func[b].func end
end
def Code
  self.class.new do
    yield
    _taskloop
    self
  end
end
```

---

## 実行例（最小構成サンプル）
### Main : rootの例と実際の再帰挙動の図解
では、いよいよ動く形でのサンプルを見ていこう

```ruby

module Merkle_tree_m
  attr_accessor :sym , :up , :func , :task
  def initialize sym: nil , up: nil , &proc
    @sym      = sym
    @up       = up
    @func     = proc
    @task     = []
  end
  def _taskloop
    task.each do | b | b.func = b.func[b].func end
  end
  def Main sym, &proc
    Task sym , &proc
    nil while not _taskloop.empty?
  end
  def Task sym = :task, &proc
    task << self.class.new(sym:sym , up:self , &proc)
  end
  def Code
    self.class.new do
      yield
      _taskloop
      self
    end
  end
end

# 拡張メソッド追加
module Merkle_tree_m_ex
  def delete
    up.task.delete self
    task.clear
  end
  def delete_lazy wait
    Task do | o | o.Code do
      if wait <= 0
        yield o if iterator?
        delete
      else
        wait -= 1
      end
  end end end
end

class Merkle_tree
  include Merkle_tree_m
  include Merkle_tree_m_ex
end

Merkle_tree.new.Main :root do |o|
  wait = 2
  o.Code do
    p "Code task"
    next unless o.task.empty?
    puts "[INIT] creating tasks under #{o.sym}"
    o.Task(:hello) { |t| t.Code { puts "Hello task run"; t } }
    o.Task(:world) { |t| t.Code { puts "World task run"; t } }
    o.delete_lazy wait do
      p "task delete_lazy #{wait}"
    end
  end
end
```

一行ずつ追っていこう
まずエントリーポイント
```ruby
Merkle_tree.new.Main :root do |o|
```
空間Aでの変数宣言、これはdelete_lazyへの引数
```ruby
wait = 2
```
空間Bでループされる部分でメッセージを表示している
```ruby
o.Code do
  p "Code task"
```
rootの持つタスクが空であれば以下が実行される
ここでは:halloタスクと:worldタスクを生成している
実行のイメージとしては creating tasks under が1回だけ表示され
Hello task run , World task run が wait+1 回分表示されるというものだ
```ruby
next unless o.task.empty?
puts "[INIT] creating tasks under #{o.sym}"
o.Task(:hello) { |t| t.Code { puts "Hello task run"; t } }
o.Task(:world) { |t| t.Code { puts "World task run"; t } }
```
タスクを遅延削除するための関数
ここでは予約をしている。
予約の方法は内部でTask / Codeによる「予約タスク」を生成するだけである
```ruby
o.delete_lazy wait do
  p "task delete_lazy #{wait}"
end
```
これを実行させると結果はこのようになる
```
"Code task"
[INIT] creating tasks under root
"Code task"
Hello task run
World task run
"Code task"
Hello task run
World task run
"Code task"
Hello task run
World task run
"task delete_lazy 2"
[Finished in 0.284s]
```
実行結果からわかるように
Tree構造のタスク、およびスケジューラーが正しく動いている


### ステップごとの「funcの書き換え」可視化
```ruby
def _taskloop
  task.each do | b | b.func = b.func[b].func end
end
```
個人的にはここの書き換えはわざわざ可視化しなくても良いと考えていて
気が向いたら書くかもしれない
自然言語で書けば、空間Aを 空間Bを返す空間Bが上書きし
1回上書きした以降は空間Bを 空間Bを返す空間Bが上書きし続けるだけである

### 1. 初回 func 呼び出し
b.func[b]          # => Codeブロック実行
      │
      ▼
### 2. func の戻り値から新しい func を取得
b.func = b.func[b].func

### 3. 次のループでは新しい func が実行される
(loop)
  ├─ func_v1
  ├─ func_v2
  ├─ func_v3 ...
func は単なるメソッドではなく「自己進化する関数オブジェクト」。
この再代入によってタスクは“生きている”ように動作する。

---

## 応用例
### Merkle_tree_m_ex
基本的なユーティリティ関数を紹介する

```ruby
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
# selfより上の一番近い nodeを返す
  def search_up sym
    return self if self.sym =~ /#{sym}/
    return up.search_up sym if up
    false
  end

  def search_up_all_sym = up ? [up.search_up_all_sym,sym]:[]

  def search_down_test o = self
    o.task.each.inject([]) do | stack , m |
      unless m.task.empty?
        next [stack , m.search_down_test(m)]
      end
      [stack , m.sym]
    end << o.sym
  end

# -------------  tree_libs
  def task_list_p(indent = 1 , first_call = true)
    # インデントを作成して、タスクの階層を見やすくする
    indent_str = '  ' * indent
    if first_call
      p "-#{self.sym} (up:#{self.TOP_SYM})"
    end
    unless task.empty?
      task.each_with_index do | o , i |
        p "#{indent_str}├─ #{o.sym} (up:#{o.up ? o.up.sym : 'none'})"
        o.task_list_p(indent + 1 , false) unless o.task.empty?
      end
    end
    p "-----------"
  end
end
```

### Lazy_delete
```ruby
def delete_lazy wait
  Task do | o | o.Code do
    if wait < 0
      yield o if iterator?
      delete
    else
      wait -= 1
    end
end end end
```

### Lazy_create
```ruby
def create_lazy wait
  Task do | o | o.Code do
    (wait < 0) ? yield(o) : (wait -= 1)
end end end
```

create_lazy(3)
   ├─ frame 1 : wait = 3
   ├─ frame 2 : wait = 2
   ├─ frame 3 : wait = 1
   └─ frame 4 : wait = 0 → タスク生成


### 将来的なタスクツリー応用
実際のところゲームのオブジェクト管理には可能であっても向かない
理由はノード単位でwork,drawなどをやるのがこの設計思想だが、
ゲームにおいてはそれだと高速化手法が取れないからである
ノード単位での高速化手法、すべてのノードをマルチスレッド化などで将来的な高速化は可能かもしれない

---

## 設計上の利点・課題
### 柔軟性・再帰的拡張性
module Merkle_tree_m を基礎とし
Merkle_tree_m_ex で拡張し
class Merkle_tree にmix-in
それから
### Merkle_tree.new.Main :root do |o|
ここから始まるコードでいかに世界を作るかという課題だが
現状この再帰構造を自由に書ける人間がほぼいない事が課題である


### 理解コスト・デバッグ性
task_tree.rbが今の形になるまで、長い時間が掛かっているため
再帰関数と再帰構造に理解がなければ手に負えない事態が来る可能性も高い


### 将来的な改善ポイント（例: 視覚化・トレース機構）
視覚化
トレース
ノード単位のマルチスレッド化


---

## まとめ
### 「Merkle_tree式」の特徴

### 全体構造

「全体の構造」総まとめ図

┌─────────────────────────────┐
│          Merkle_tree         │
│     ├─ task[]                │
│     ├─ func (動的Proc)       │
│     ├─ up (親参照)          │
│     └─ sym (識別子)          │
└─────────────────────────────┘
          │
          ▼
   ┌────────────┐
   │ _taskloop  │ ← 全ノード巡回
   └────────────┘
          │
          ▼
   func[b].func により func 更新
          │
          ▼
   再帰呼び出し → 次ループ


これで一通りの説明は終わりにしたい


最後に
### Merkle_tree.new.Main :root do |o|
実際にゲームのシーン管理で使っているツリーを乗せる

```ruby
class Merkle_tree
  include Merkle_tree_m
  include Merkle_tree_m_ex
  include DEBUG_CODE_m
  include Node_add
  # def initialize **_
  #   super
  # end
  # lazy initialization
  def Flandoll = @Flandoll ||= []
  def Scarlet  = @Scarlet ||= Hash.new { |h, k| h[k] = [] }
end

Merkle_tree.new.Main :top_node do |o|
  o.Code do
    if o.task.empty? then o.Flandoll << File.basename($0,".*").to_sym  end
    -> &b do lambda do |f|
        lambda{|x|lambda{|y| f[x[x]] [y]}}[
          lambda{|x|lambda{|y| f[x[x]] [y]}}]
      end[ lambda{|f|lambda{|n| b[n , &f] }}]
    end.yield do | o , rb , &f |
      case o.Flandoll.shift
      when nil then true
      when -> rb do o.Object_create rb end
      when -> rb do o.Scene_loading rb end
      when -> rb do
          o.Task :"__#{rb}_task" do |o|
            o.Code do
              o.Main :"#{rb}" do |o|
                o.Scene_create rb
                o.Code do
                  f[ [o , rb] ]
                end # code
              end # main
            end # co
          end # tas
        end #lm
      end # while not o.Flandoll.empty? # case
    end[ [ o , nil ] ] # recursion
  end # code do
end # main

```

簡潔に書くにはYコンビネータが非常に重要で
自己ブロックを再帰呼び出しで
### o.Scene_create rb
シーンジェネレータのインスタンス生成もYコンビネータで
```ruby
end.yield do | ( o , rb ) , &f |
```
```ruby
o.Code do
  f[ [o , rb] ]
end # code
```
このようにlambdaブロックそのものを再帰する
このYコンビネータの活用によって
全てのシーンの生成を担えるのでMerkle_scene.newの呼び出しの記述はゲームのソースコード中ここだけである

I don't know what you're saying anymore ...

例えば Merkle_scene.new の中で生成されるゲームのメッセージループの各シーンの中において
```ruby
loop do |o|
  Window.sync
  Window.update
  exit if Input.update
  if 条件
    o.Frandre << :scene_name # scene_name.rbを loadまたはevalするメッセージキュー
  end
end
```
このようにするだけで
```ruby
case o.Frandre.shift
```
この中で :scene_name が読み取られ

```ruby
when -> rb do o.Scene_loading end
```
一度loadingシーンを挟みつつ （これはゲーム設計により省略可能)

次のwhen節へ入りタスクの生成およびループの生成が実行される子タスクの実行
```ruby
when -> rb do
    o.Task :"__#{rb}_task" do |o|
      o.Code do
        o.Main :"#{rb}" do |o|
          Merkle_scene.new o , rb
          o.Code do
            f[ [o , rb] ]
          end # code
        end # main
      end # co
    end # tas
  end #lm
```

ここは完全に魔術化しているが、ようは子タスクから親タスクをdeleteするような場面で、更なるTaskのネストが必要だからである
```ruby
o.Task :"__#{rb}_task" do |o|
  o.Code do
    o.Main :"#{rb}" do |o|
```
将来的にネストが小さくなる可能性はあり得るが
再帰構造のデバッグは容易ではないので、現時点ではこういうコードになっている


### 終わりに

これでMerkle_treeの説明は終わり
興味を持ったらこの木を成長させてみて

“How to Run”
ruby sample.rb
