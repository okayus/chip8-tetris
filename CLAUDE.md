# chip8-tetris

chip8-lang (自作関数型言語) で書かれた CHIP-8 TETRIS。

## 概要

- ソース: `src/tetris.ch8l`
- ROM 出力: `src/tetris.ch8` (コンパイラが同ディレクトリに出力)
- 設計: `docs/DESIGN.md`
- 関数型設計の経緯: `docs/FUNCTIONAL_TETRIS.md`

## コマンド

```bash
# コンパイル
cargo run --manifest-path ../chip8-lang/Cargo.toml -- src/tetris.ch8l
```

## 開発フロー

- main ブランチへの直プッシュは禁止
- Phase ごとにブランチを作成し、PR を通じてマージ
- ブランチ命名: `phase/{番号}-{短い説明}`
