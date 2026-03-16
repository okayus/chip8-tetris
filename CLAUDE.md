# chip8-tetris

chip8-lang (自作関数型言語) で書かれた CHIP-8 TETRIS。

## 概要

- ソース: `src/tetris.ch8l`
- ROM 出力: `rom/tetris.ch8`
- 設計: `docs/DESIGN.md`
- コンパイラ: `../chip8-lang/` (chip8-lang)
- エミュレータ: `../chip8-ts/` (chip8-ts)

## コマンド

```bash
# コンパイル
cargo run --manifest-path ../chip8-lang/Cargo.toml -- src/tetris.ch8l

# エミュレータで確認 (chip8-ts)
# rom/tetris.ch8 を chip8-ts の public/roms/ にコピーして起動
```

## 開発フロー

- main ブランチへの直プッシュは禁止
- Phase ごとにブランチを作成し、PR を通じてマージ
- ブランチ命名: `phase/{番号}-{短い説明}`
