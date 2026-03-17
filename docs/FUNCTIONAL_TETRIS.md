# 関数型 TETRIS — 理想形の設計

現在の TETRIS (131行) を、言語機能が十分にあると仮定して関数型スタイルで書き直すとどうなるか。

## 現状の問題

```
-- main が 80 行の手続きループ
-- ミュータブルな変数 8 個を直接操作
-- erase → move → draw → check → revert の手順がインライン展開
-- 左移動と右移動がほぼ同じコードの重複
```

## 理想形: 不変データ + 純粋関数 + 状態遷移

### 必要な言語機能

| 機能 | 用途 | 現状 |
|------|------|------|
| struct / record | ゲーム状態をまとめる | なし (変数 8 個がバラバラ) |
| enum + match | ピース種別・入力・状態遷移 | match のみあり |
| タプル / 多値返却 | 関数から複数値を返す | なし |
| クロージャ / 高階関数 | try_move を汎用化 | なし |
| パイプ演算子 | 状態変換の連鎖 | あるが struct がないと活かせない |
| 再帰 (末尾最適化) | ゲームループを再帰で表現 | なし (loop のみ) |

### 理想のコード

```
-- ============================================================
-- 型定義
-- ============================================================

enum Piece { I, O, T, S, Z, L, J }

enum Input { Left, Right, Drop, None }

struct Pos { x: u8, y: u8 }

struct GameState {
  piece: Piece,
  pos: Pos,
  score: u8,
  speed: u8,
}

-- ============================================================
-- 純粋関数 (副作用なし)
-- ============================================================

-- 新しい位置を計算 (元のデータは不変)
fn move_pos(pos: Pos, dx: u8, dy: u8) -> Pos {
  Pos { x: pos.x + dx, y: pos.y + dy }
}

-- 入力をポジション変化に変換
fn input_to_delta(input: Input) -> Pos {
  match input {
    Input::Left  => Pos { x: -2, y: 0 },
    Input::Right => Pos { x: 2, y: 0 },
    Input::Drop  => Pos { x: 0, y: 2 },
    Input::None  => Pos { x: 0, y: 0 },
  }
}

-- ランダムなピースを生成
fn random_piece() -> Piece {
  match random(6) {
    0 => Piece::I,
    1 => Piece::O,
    2 => Piece::T,
    3 => Piece::S,
    4 => Piece::Z,
    5 => Piece::L,
    6 => Piece::J,
  }
}

-- 着地後の新しいゲーム状態を計算
fn on_land(state: GameState) -> GameState {
  GameState {
    piece: random_piece(),
    pos: Pos { x: 28, y: 0 },
    score: state.score + 1,
    speed: if state.speed > 8 { state.speed - 3 } else { state.speed },
  }
}

-- ============================================================
-- 描画関数 (副作用あり、だが状態を変更しない)
-- ============================================================

fn draw_piece(piece: Piece, pos: Pos) -> bool {
  match piece {
    Piece::I => draw(piece_i, pos.x, pos.y),
    Piece::O => draw(piece_o, pos.x, pos.y),
    Piece::T => draw(piece_t, pos.x, pos.y),
    Piece::S => draw(piece_s, pos.x, pos.y),
    Piece::Z => draw(piece_z, pos.x, pos.y),
    Piece::L => draw(piece_l, pos.x, pos.y),
    Piece::J => draw(piece_j, pos.x, pos.y),
  }
}

-- ============================================================
-- 移動試行 (erase → draw → revert のパターンを関数化)
-- ============================================================

-- ピースの移動を試行し、成功なら新 Pos を、衝突なら元 Pos を返す
fn try_move(piece: Piece, old: Pos, new: Pos) -> Pos {
  draw_piece(piece, old);             -- erase
  let col: bool = draw_piece(piece, new);  -- draw at new
  if col {
    draw_piece(piece, new);            -- undo
    draw_piece(piece, old);            -- restore
    old                                -- 元の位置を返す (移動失敗)
  } else {
    new                                -- 新しい位置を返す (移動成功)
  }
}

-- ============================================================
-- 入力読み取り (副作用 → 純粋な値への変換)
-- ============================================================

fn read_input() -> Input {
  if is_key_pressed(4) { return Input::Left; };
  if is_key_pressed(6) { return Input::Right; };
  if is_key_pressed(8) { return Input::Drop; };
  Input::None
}

-- ============================================================
-- ゲームループ (状態遷移の再帰)
-- ============================================================

-- 1フレームの処理: 現在の状態 → 次の状態
fn tick(state: GameState) -> GameState {
  let input: Input = read_input();

  -- 入力処理: 左右移動
  let pos: Pos = match input {
    Input::Left  => if state.pos.x > 2 {
      try_move(state.piece, state.pos, move_pos(state.pos, -2, 0))
    } else { state.pos },
    Input::Right => if state.pos.x < 56 {
      try_move(state.piece, state.pos, move_pos(state.pos, 2, 0))
    } else { state.pos },
    Input::Drop  => { set_delay(1); state.pos },
    Input::None  => state.pos,
  };

  -- 落下処理
  if delay() == 0 {
    let new_pos: Pos = try_move(state.piece, pos, move_pos(pos, 0, 2));
    if new_pos == pos {
      -- 移動失敗 = 着地
      set_sound(2);
      let next: GameState = on_land(GameState { ..state, pos: pos });
      update_score_display(next.score);
      let col: bool = draw_piece(next.piece, next.pos);
      if col {
        gameover_screen(next.score);
        -- ゲーム終了 (再帰しない)
      };
      set_delay(next.speed);
      next
    } else {
      set_delay(state.speed);
      GameState { ..state, pos: new_pos }
    }
  } else {
    GameState { ..state, pos: pos }
  }
}

-- ゲームループ: tick を繰り返す (末尾再帰で最適化)
fn game_loop(state: GameState) -> () {
  let next: GameState = tick(state);
  game_loop(next)   -- 末尾再帰 → ループに最適化
}

-- ============================================================
-- エントリーポイント
-- ============================================================

fn main() -> () {
  title_screen();
  init_field();

  let initial: GameState = GameState {
    piece: Piece::T,
    pos: Pos { x: 28, y: 0 },
    score: 0,
    speed: 25,
  };

  draw_piece(initial.piece, initial.pos);
  draw_digit(initial.score, 56, 0);
  set_delay(initial.speed);

  game_loop(initial);
}
```

## 実装結果と CHIP-8 の壁

### 実現できたもの (PR #12)

| 機能 | 活用方法 |
|------|---------|
| `Piece` enum | `u8` → 型安全な `Piece::I`, `Piece::T`, ... |
| match on enum | `match p { Piece::I => draw(piece_i, x, y), ... }` |
| `random_enum(Piece)` | ランダムピース生成を型安全に |
| `try_move() -> bool` | erase/draw/revert パターンを関数化 |
| レジスタ退避 | main → try_move → draw_piece の 3 段呼び出し |

ROM: **2217 → 673 bytes (70% 削減)**, ソース: **244 → 127 行 (48% 削減)**

### CHIP-8 の 15 レジスタ制約で実現できないもの

| 理想形 | 制約 | 現実的な代替 |
|--------|------|-------------|
| `Pos` struct を関数引数に | struct 2 フィールド + 他の引数 + save/restore でレジスタオーバーフロー | スカラー `u8` 引数 (px, py) |
| `GameState` struct で状態管理 | 5+ フィールド = 5+ レジスタ。tick 内の関数呼び出しで上限超過 | 個別のミュータブル変数 |
| 末尾再帰 `game_loop(state)` | GameState が 5 レジスタを引数で消費し、ループ内の関数呼び出しで上限超過 | `loop { ... break; }` |
| `Input` enum + match で入力分岐 | 追加のローカル変数がレジスタを圧迫 | `if is_key_pressed(N)` の直接呼び出し |

### 根本原因

CHIP-8 は汎用レジスタが V0-VE の **15 本** しかない。
関数呼び出し時の caller-save (FX55/FX65) でローカル変数をメモリに退避する際、
退避する変数数 + 引数 + テンポラリ + 戻り値の合計が 15 を超えると使えない。

struct を引数に渡すとフィールド数分のレジスタを消費するため、
2 フィールドの `Pos` ですら複数引数と組み合わせると上限に達する。

### 結論

CHIP-8 のレジスタ制約の中で、関数型スタイルは **「enum + match + 関数抽出」** のレベルまでは実用的。
**「不変 struct + 末尾再帰」** は理想的だが、15 レジスタの壁により CHIP-8 では実現困難。
