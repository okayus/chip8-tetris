# tetris.ch8l コード解説

chip8-lang で書かれた CHIP-8 TETRIS の全コードを、処理の流れに沿って解説する。
後半では、ゲーム開始時とキー入力時の具体的な値トレースを付記する。

---

## 目次

1. [型定義](#1-型定義)
2. [スプライトデータ](#2-スプライトデータ)
3. [ピース描画](#3-ピース描画-draw_piece)
4. [移動・回転](#4-移動回転)
5. [画面ユーティリティ](#5-画面ユーティリティ)
6. [ライン消去](#6-ライン消去)
7. [ゲームロジック](#7-ゲームロジック)
8. [落下処理](#8-落下処理-handle_fall)
9. [回転ロック](#9-回転ロック)
10. [入力処理](#10-入力処理-handle_input)
11. [ティック・ゲームループ](#11-ティックゲームループ)
12. [エントリーポイント](#12-エントリーポイント-main)
13. [値トレース: ゲーム開始](#13-値トレース-ゲーム開始)
14. [値トレース: キー入力で左移動](#14-値トレース-キー入力で左移動)
15. [値トレース: 回転](#15-値トレース-回転)
16. [値トレース: 自動落下と着地](#16-値トレース-自動落下と着地)

---

## 1. 型定義

```
enum Piece { I, O, T, S, Z, L, J }

struct GameState { piece: Piece, rot: u8, x: u8, y: u8, score: u8, speed: u8 }
```

- `Piece`: 7 種のテトロミノを表す enum。コンパイラが 0〜6 の整数にマッピングする
- `GameState`: ゲームの全状態を 6 フィールドの不変 struct で管理する
  - `piece`: 現在のピース種
  - `rot`: 回転状態 (0〜3)
  - `x`, `y`: ピースの描画座標 (ピクセル単位)
  - `score`: 現在スコア
  - `speed`: 落下速度 (delay timer にセットする値。小さいほど速い)

## 2. スプライトデータ

```
let i_r0: sprite(4) = [0x40, 0x40, 0x40, 0x40];   -- I-piece 縦
let i_r1: sprite(4) = [0x00, 0xF0, 0x00, 0x00];   -- I-piece 横
...
let wall: sprite(1) = [0x80];                       -- 壁 1px
let floor: sprite(1) = [0xFF];                      -- 床 8px
let flag: sprite(1) = [0x80];                       -- 判定用 1px フラグ
```

各ピースは `sprite(4)` (4 バイト = 8px 幅 × 4 行) で定義。1px = 1 セル。
回転ごとに別スプライトを用意する (I/S/Z は 2 種、O は 1 種、T/L/J は 4 種)。

**ビットパターンの読み方** (T-piece rot=0 の例):

```
t_r0 = [0x40, 0xE0, 0x00, 0x00]

0x40 = 0100 0000  →  .#..
0xE0 = 1110 0000  →  ###.
0x00 = 0000 0000  →  ....
0x00 = 0000 0000  →  ....
```

`wall` と `flag` は同じ `[0x80]` (左端 1px) だが、用途が異なる。`wall` は壁描画、`flag` はピクセル単位の状態読み取り (回転ロック、ライン消去判定) に使う。

## 3. ピース描画 (`draw_piece`)

```
fn draw_piece(piece: Piece, rot: u8, x: u8, y: u8) -> bool {
  match piece {
    Piece::I => match rot { 0 => draw(i_r0, x, y), 1 => draw(i_r1, x, y), ... },
    Piece::O => draw(o_r0, x, y),
    ...
  }
}
```

**nested match**: piece で外側分岐 → rot で内側分岐 → 対応するスプライトを `draw` する。
コンパイラは `JpV0 (BNNN)` によるジャンプテーブルで O(1) ディスパッチに最適化する。

`draw` は CHIP-8 の `DRW` 命令 (XOR 描画) を呼び、戻り値 `bool` は衝突フラグ (VF)。
既存ピクセルと重なると `true` を返す。

## 4. 移動・回転

### `try_move`: 移動試行

```
fn try_move(p: Piece, r: u8, ox: u8, oy: u8, nx: u8, ny: u8) -> bool {
  draw_piece(p, r, ox, oy);               -- (1) 現在位置を XOR で消去
  let col: bool = draw_piece(p, r, nx, ny); -- (2) 新位置に描画、衝突判定
  if col {
    draw_piece(p, r, nx, ny);              -- (3a) 衝突: 新位置を消去
    draw_piece(p, r, ox, oy);              -- (3b) 元位置を復元
    false                                   -- 移動失敗
  } else {
    true                                    -- 移動成功 (新位置に描画済み)
  }
}
```

XOR の性質を利用: 同じ位置に 2 回描画すると元に戻る。

引数がスカラー 6 個なのは、`Pos` struct を渡すとレジスタ超過するため。

### `try_rotate`: 回転試行

```
fn try_rotate(p: Piece, r: u8, x: u8, y: u8) -> u8 {
  let nr: u8 = if r == 3 { 0 } else { r + 1 };
  draw_piece(p, r, x, y);                  -- 現在の回転を消去
  let col: bool = draw_piece(p, nr, x, y); -- 新回転で描画
  if col {
    draw_piece(p, nr, x, y);               -- 衝突: 新回転を消去
    draw_piece(p, r, x, y);                -- 元の回転を復元
    r                                       -- 回転失敗、元の rot を返す
  } else {
    nr                                      -- 回転成功、新しい rot を返す
  }
}
```

`try_move` と同じ erase → try → revert パターン。戻り値は新しい rot 値。

## 5. 画面ユーティリティ

### `draw_walls`

```
fn draw_walls() -> () {
  let y: u8 = 0;
  loop {
    if y > 30 { break; };
    draw(wall, 0, y);     -- 左壁 (x=0)
    draw(wall, 11, y);    -- 右壁 (x=11)
    y = y + 1;
  };
  draw(floor, 0, 31);     -- 床 (y=31, 左半分)
  draw(floor, 8, 31);     -- 床 (y=31, 右半分)
}
```

プレイフィールド: x=0 (左壁)、x=1〜10 (10 列のピース移動領域)、x=11 (右壁)、y=31 (床)。

### `title_screen` / `gameover_screen`

タイトル: テトロミノスプライトを装飾的に配置 → `wait_key()` でキー待ち。
ゲームオーバー: サウンド再生 → スコア表示 → キー待ち。

## 6. ライン消去

### `clear_lines`: ライン検出と消去

```
fn clear_lines(base_y: u8) -> u8
```

ピースが着地した `base_y` から下方向 3 行 (`base_y` 〜 `base_y + 3`) を走査する。

**1 行の充填判定**: x=1〜10 の各ピクセルに `flag` (1px) を XOR 描画。VF=1 (衝突) なら ON、VF=0 なら OFF。判定後すぐ再描画して元に戻す。全 10 ピクセルが ON なら `full = true`。

### `shift_rows_down`: 行の下方シフト

```
fn shift_rows_down(cy: u8) -> ()
```

消去対象行 `cy` から上に向かって 1 行ずつ下にコピーする。

**ピクセルコピーの仕組み** (各 x について):
1. `draw(flag, rx, row)` → VF で dest ピクセルの状態を読み取り
2. VF=0 (OFF) なら再描画で元に戻す。VF=1 (ON) ならそのまま (XOR で OFF にした)
3. `draw(flag, rx, src)` → VF で src ピクセルの状態を読み取り、再描画で復元
4. src が ON なら dest に描画

最上行 (y=0) は常にクリアする (降ろすべき行がないため)。

## 7. ゲームロジック

### `spawn_piece`: 新ピース生成

```
fn spawn_piece(np: Piece, sc: u8, sp: u8) -> GameState {
  let col: bool = draw_piece(np, 0, 4, 0);
  if col {
    GameState { piece: np, rot: 0, x: 4, y: 0, score: sc, speed: 0 }  -- ゲームオーバー
  } else {
    let nsp: u8 = next_speed(sp);
    set_delay(nsp);
    GameState { piece: np, rot: 0, x: 4, y: 0, score: sc, speed: nsp }
  }
}
```

初期位置 `(4, 0)` に描画。衝突すれば `speed: 0` を返す → `game_loop` が検出してゲームオーバー。

### `land_and_spawn`: 着地 → ライン消去 → 新ピース

```
fn land_and_spawn(state: GameState) -> GameState {
  ...
  let lines: u8 = clear_lines(y);
  let new_sc: u8 = sc + 1 + lines;       -- 着地 1 点 + 消去行数
  update_score(sc, new_sc);
  let np: Piece = random_enum(Piece);
  spawn_piece(np, new_sc, sp)
}
```

### `next_speed`: 速度漸増

```
fn next_speed(speed: u8) -> u8 {
  if speed > 5 { speed - 2 } else { speed }
}
```

着地のたびに speed が 2 減少 (= タイマー間隔短縮 = 落下高速化)。最小 5 で下げ止まり。

## 8. 落下処理 (`handle_fall`)

```
fn handle_fall(state: GameState) -> GameState {
  ...
  let ny: u8 = y + 1;
  let ok: bool = try_move(p, r, x, y, x, ny);
  if ok {
    set_delay(sp);
    GameState { piece: p, rot: r, x: x, y: y + 1, score: sc, speed: sp }
  } else {
    state |> land_and_spawn
  }
}
```

`delay() == 0` のとき呼ばれる。`try_move` で 1px 下に移動を試行。
- 成功: y+1 の新 GameState を返し、delay を再セット
- 失敗: 着地 → `land_and_spawn` をパイプで呼ぶ

## 9. 回転ロック

回転キーの長押しで連続回転するのを防ぐデバウンス機構。画面右上端 `(63, 0)` のピクセルをフラグとして使う。

```
fn is_rot_locked() -> bool    -- (63,0) のピクセルを読み、元に戻す
fn set_rot_lock() -> ()       -- (63,0) を確実に ON にする
fn clear_rot_lock() -> ()     -- (63,0) を確実に OFF にする
```

仕組み: `draw(flag, 63, 0)` で XOR 描画 → VF が 1 なら元々 ON (ロック中)、0 なら OFF。
判定後にピクセルを元に戻す (非破壊読み取り)。

- 回転キー押下中: `set_rot_lock()` してから回転 → 次フレームでは `is_rot_locked() == true` なので無視
- キーを離した時: `clear_rot_lock()` → 再度押せる

## 10. 入力処理 (`handle_input`)

```
fn handle_input(state: GameState) -> GameState {
  ...
  if is_key_pressed(4) {             -- Q: 左移動
    if x > 0 {
      let ok: bool = try_move(p, r, x, y, x - 1, y);
      if ok { GameState { ..., x: x - 1, ... } }
      else { state }
    } else { state }
  } else { if is_key_pressed(6) {    -- R: 右移動
    let ok: bool = try_move(p, r, x, y, x + 1, y);
    if ok { GameState { ..., x: x + 1, ... } }
    else { state }
  } else { if is_key_pressed(5) {    -- W: 回転
    if is_rot_locked() { state }
    else { set_rot_lock(); do_rotate(state) }
  } else {
    clear_rot_lock();
    if is_key_pressed(8) { set_delay(1); };  -- S: ソフトドロップ
    state
  }}}
}
```

優先順位: 左 > 右 > 回転 > ドロップ。同時押しは上位が優先。

ソフトドロップは `set_delay(1)` で delay timer をほぼ 0 にし、次ティックで `handle_fall` が呼ばれるようにする。

## 11. ティック・ゲームループ

### `tick`

```
fn tick(state: GameState) -> GameState {
  if delay() == 0 {
    handle_fall(state)
  } else {
    handle_input(state)
  }
}
```

delay timer が 0 なら落下、それ以外なら入力処理。
CHIP-8 の delay timer はエミュレータが自動デクリメントする (Extended モードで 50 命令/tick)。

### `game_loop`

```
fn game_loop(state: GameState) -> () {
  if state.speed == 0 {
    gameover_screen(state.score)
  } else {
    state |> tick |> game_loop
  }
}
```

`speed == 0` はゲームオーバーの合図 (`spawn_piece` が衝突時にセット)。
末尾再帰なのでコンパイラが `loop` に最適化する (スタック消費なし)。

## 12. エントリーポイント (`main`)

```
fn main() -> () {
  title_screen();
  clear();
  draw_walls();

  let initial: GameState = GameState {
    piece: Piece::T, rot: 0, x: 4, y: 0, score: 0, speed: 60,
  };

  draw_digit(initial.score, 56, 0);
  draw_piece(initial.piece, initial.rot, initial.x, initial.y);
  set_delay(initial.speed);

  game_loop(initial);
}
```

1. タイトル画面表示 → キー待ち
2. 画面クリア → 壁描画
3. 初期状態: T-piece、位置 (4,0)、速度 60
4. スコア "0" 表示、最初のピースを描画、delay=60 にセット
5. ゲームループ開始

---

## 13. 値トレース: ゲーム開始

`main()` が呼ばれてからゲームループが回り始めるまでの値の変化。

### ステップ 1: `title_screen()`

```
clear()          -- 画面全クリア (64x32 = 0)
draw(t_r0, 4, 4) -- T-piece スプライトを (4,4) に描画 (装飾)
draw(i_r1, 14, 5) -- I-piece を (14,5) に
...
wait_key()        -- キー入力待ち (ブロッキング)
```

### ステップ 2: 初期化

```
clear()           -- タイトル画面を消去
draw_walls()      -- 壁描画:
                  --   x=0, y=0..30  → 左壁 (31 個の 1px ドット)
                  --   x=11, y=0..30 → 右壁
                  --   (0,31) + (8,31) → 床 (16px 幅)
```

画面状態 (プレイフィールド部分):
```
x: 0 1 2 3 4 5 6 7 8 9 10 11
   # . . . . . . . . . .  #   y=0
   # . . . . . . . . . .  #   y=1
   ...
   # . . . . . . . . . .  #   y=30
   # # # # # # # # # # #  #   y=31 (床)
```

### ステップ 3: 初期 GameState 作成

```
initial = GameState {
  piece: Piece::T,    -- T ピース
  rot: 0,             -- 回転なし
  x: 4,               -- 左壁(0) から 4px 右
  y: 0,               -- 最上段
  score: 0,
  speed: 60,          -- delay timer = 60 (最も遅い)
}
```

### ステップ 4: 初期描画

```
draw_digit(0, 56, 0)          -- スコア "0" を (56,0) に表示
draw_piece(Piece::T, 0, 4, 0) -- T-piece rot=0 を (4,0) に描画
```

T-piece rot=0 のスプライト `t_r0 = [0x40, 0xE0, 0x00, 0x00]`:
```
x: 0 1 2 3 4 5 6 7 8 9 10 11
   # . . . . # . . . . .  #   y=0  (0x40 の bit6 → x=4+1=5 に dot)
   # . . . # # # . . . .  #   y=1  (0xE0 の bit7,6,5 → x=4,5,6 に dot)
   # . . . . . . . . . .  #   y=2
```

### ステップ 5: タイマーセット → ループ開始

```
set_delay(60)    -- delay timer = 60
game_loop(initial)
```

ここから `game_loop` → `tick` → `handle_input` or `handle_fall` のサイクルが始まる。

delay timer は 60 > 0 なので、最初のティックでは `handle_input` が呼ばれる。

---

## 14. 値トレース: キー入力で左移動

**状況**: T-piece が `(4, 3)` にいる。プレイヤーが Q キー (key 4) を押す。

### 現在の GameState

```
state = { piece: T, rot: 0, x: 4, y: 3, score: 0, speed: 58 }
```

### `tick(state)`

```
delay() → 42 (> 0)  -- タイマーまだカウント中
→ handle_input(state)
```

### `handle_input(state)`

```
p=T, r=0, x=4, y=3, sc=0, sp=58

is_key_pressed(4) → true (Q キー押下中)
x=4 > 0 → true (左壁チェック OK)
→ try_move(T, 0, 4, 3, 3, 3) を呼ぶ
```

### `try_move(T, 0, ox=4, oy=3, nx=3, ny=3)`

```
(1) draw_piece(T, 0, 4, 3)    -- 現在位置 (4,3) の T-piece を XOR 消去
    → t_r0 スプライトが (4,3) から消える
    → VF は無視 (戻り値未使用)

(2) draw_piece(T, 0, 3, 3)    -- 新位置 (3,3) に T-piece を XOR 描画
    → t_r0 スプライトが (3,3) に出現
    → col = false (壁や他ピースと重なっていない)

col == false → true を返す (移動成功)
```

### 結果

```
ok = true
→ GameState { piece: T, rot: 0, x: 3, y: 3, score: 0, speed: 58 }
```

画面上のピースが 1px 左にずれた:

```
移動前 (x=4):               移動後 (x=3):
x: 0 1 2 3 4 5 6 7          x: 0 1 2 3 4 5 6 7
   # . . . . # . .  y=3        # . . . # . . .  y=3
   # . . . # # # .  y=4        # . . # # # . .  y=4
```

### 壁際の左移動が失敗するケース

**状況**: ピースが `x=1` にいて Q を押した場合。

```
handle_input: x=1 > 0 → true
→ try_move(T, 0, 1, 3, 0, 3)

(1) draw_piece(T, 0, 1, 3) -- (1,3) の T を消去
(2) draw_piece(T, 0, 0, 3) -- (0,3) に描画 → t_r0 の bit7 が x=0 (左壁位置)
    → 壁ピクセルと衝突 → col = true

col == true:
(3a) draw_piece(T, 0, 0, 3) -- (0,3) を消去 (元に戻す)
(3b) draw_piece(T, 0, 1, 3) -- (1,3) を復元
→ false

ok = false → state をそのまま返す (位置変わらず)
```

---

## 15. 値トレース: 回転

**状況**: T-piece が `(4, 5)` にいて rot=0。プレイヤーが W キー (key 5) を初めて押す。

### 現在の GameState

```
state = { piece: T, rot: 0, x: 4, y: 5, score: 0, speed: 58 }
```

### `handle_input(state)`

```
is_key_pressed(4) → false
is_key_pressed(6) → false
is_key_pressed(5) → true (W キー押下)

is_rot_locked() → false (初回押下、(63,0) は OFF)
→ set_rot_lock()   -- (63,0) を ON にする
→ do_rotate(state)
```

### `do_rotate(state)`

```
p=T, r=0, x=4, y=5
→ try_rotate(T, 0, 4, 5)
```

### `try_rotate(T, 0, 4, 5)`

```
nr = if 0 == 3 { 0 } else { 0 + 1 } = 1

(1) draw_piece(T, 0, 4, 5) -- rot=0 (t_r0) を消去
    t_r0: .#..
          ###.

(2) draw_piece(T, 1, 4, 5) -- rot=1 (t_r1) を描画
    t_r1: .#..
          ##..
          .#..
    → col = false (空きスペースに収まる)

col == false → nr = 1 を返す
```

### 結果

```
nr=1, r=0 → nr != r
→ GameState { piece: T, rot: 1, x: 4, y: 5, score: 0, speed: 58 }
```

画面の変化:
```
回転前 (rot=0, t_r0):       回転後 (rot=1, t_r1):
   . # . .  y=5                . # . .  y=5
   # # # .  y=6                # # . .  y=6
   . . . .  y=7                . # . .  y=7
```

### 次フレームで W を押し続けた場合

```
is_key_pressed(5) → true
is_rot_locked() → true ((63,0) が ON)
→ state をそのまま返す (回転しない)
```

### W を離した次のフレーム

```
is_key_pressed(4) → false
is_key_pressed(6) → false
is_key_pressed(5) → false
→ clear_rot_lock()   -- (63,0) を OFF
→ is_key_pressed(8) をチェック → 通常は false
→ state を返す
```

---

## 16. 値トレース: 自動落下と着地

**状況**: T-piece (rot=0) が `(4, 28)` にいて、delay timer がちょうど 0 になった。
床は y=31。

### 現在の GameState

```
state = { piece: T, rot: 0, x: 4, y: 28, score: 3, speed: 54 }
```

### `tick(state)`

```
delay() → 0
→ handle_fall(state)
```

### `handle_fall` — 落下成功 (y=28 → 29)

```
p=T, r=0, x=4, y=28, ny=29
try_move(T, 0, 4, 28, 4, 29) → true (空きあり)

set_delay(54)
→ GameState { piece: T, rot: 0, x: 4, y: 29, score: 3, speed: 54 }
```

### 次のティック: `handle_fall` — 着地 (y=29 → 30 失敗)

```
state = { piece: T, rot: 0, x: 4, y: 29, score: 3, speed: 54 }
delay() → 0
→ handle_fall(state)

ny = 30
try_move(T, 0, 4, 29, 4, 30):
  (1) draw_piece(T, 0, 4, 29) -- 消去
  (2) draw_piece(T, 0, 4, 30) -- 描画試行
      t_r0 の 2 行目 (0xE0 = ###) が y=31 (床) と衝突
      → col = true
  (3a) draw_piece(T, 0, 4, 30) -- 新位置を消去
  (3b) draw_piece(T, 0, 4, 29) -- 元位置を復元
→ false

ok == false → state |> land_and_spawn
```

### `land_and_spawn(state)`

```
sc=3, sp=54, y=29
set_sound(2)                       -- 着地音

clear_lines(29):
  ry = min(29+3, 30) = 30          -- y=30 から上に走査
  y=30: x=1..10 を flag で走査 → 全部 ON? → full = true → shift_rows_down(30)
        lines = 1
  y=30 (再チェック): 上の行が降りてきた → full = false
  y=29: 走査 → full = false
  → lines = 1

new_sc = 3 + 1 + 1 = 5             -- 着地1点 + 消去1行
update_score(3, 5)                  -- スコア表示を 3 → 5 に更新

np = random_enum(Piece)             -- 例: Piece::S
→ spawn_piece(S, 5, 54)
```

### `spawn_piece(S, 5, 54)`

```
draw_piece(S, 0, 4, 0)             -- S-piece を (4,0) に描画
→ col = false (最上段が空いている)

nsp = next_speed(54) = 52           -- 速度アップ
set_delay(52)
→ GameState { piece: S, rot: 0, x: 4, y: 0, score: 5, speed: 52 }
```

### ゲームオーバーのケース

もし `spawn_piece` で衝突が起きた場合:

```
draw_piece(S, 0, 4, 0) → col = true (積み上がったブロックと衝突)
→ GameState { piece: S, rot: 0, x: 4, y: 0, score: 5, speed: 0 }
```

`game_loop` が `speed == 0` を検出:

```
game_loop(state):
  state.speed == 0 → true
  → gameover_screen(5)   -- スコア 5 を表示して終了
```
