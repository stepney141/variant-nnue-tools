## 66shogi Implementation Notes

### Objectives
- Provide engine-level support for the staged placement sequence unique to 66shogi without breaking existing shogi-style variants.
- Enforce the "either rook or bishop" rule mechanically so that users don’t have to self-police their setup.
- Keep the configuration surface variant-driven: everything should be expressible from `variants.ini` rather than hardcoding behavior in the search logic.

### Key Design Decisions
1. **Generic Setup Phase Hooks**
   - Added `setupDrops`, `setupMustDrop`, per-color `setupDropRegion`, and `setupDropPieceTypes` to `Variant`. This lets any variant specify a temporary drop-only phase, its valid squares, and which pockets are mandatory before regular play begins.
   - `StateInfo` now tracks `setupDropsActive` so legality checks can force drops only during the setup window, avoiding conflicts with normal shogi drops after the phase ends.

2. **Piece Choice Groups**
   - Introduced `PieceChoiceGroup` arrays in `Variant` and parser support for `pieceChoiceGroup*` keys. Each group defines a piece set, a maximum count, and whether it applies only during setup.
   - During setup, move generation consults the relevant group to ensure that, for example, dropping a rook consumes the shared quota and prevents a subsequent bishop drop (and vice versa).

3. **Automatic Cleanup After Setup**
   - Once all required drops are performed, any leftover pocket pieces belonging to fulfilled choice groups are removed so later captures don’t resurrect the unused option.
   - NNUE dirty-piece bookkeeping was extended so that these implicit removals keep the accumulator in sync.

4. **Variant Definition**
   - Added `sixty_six_shogi_variant()` in `src/variant.cpp` and a matching `[66shogi:shogi]` section in `variants.ini` that leverage the new knobs: 6×6 board, custom start FEN with pockets, setup drop regions (rank 1 for White, rank 6 for Black), and `pieceChoiceGroup = options=rb limit=1 required=true`.

### File Touchpoints
- `src/variant.h / src/variant.cpp`: new struct members, helpers, and the concrete 66shogi variant.
- `src/parser.h / src/parser.cpp`: parsing logic for setup fields and piece-choice groups.
- `src/position.h / src/position.cpp`: enforcement of setup-phase drops, choice-group accounting, and automatic cleanup.
- `src/variants.ini`: declarative definition of the 66shogi ruleset using the newly introduced options.

### Result
The engine can now run 66shogi end-to-end: players alternate forced drops on their home ranks, must choose exactly one of rook/bishop, and transition seamlessly into standard shogi play once all six pieces are deployed. The infrastructure is generic enough for future placement-based variants.

---

## 66shogi 実装メモ（日本語）

### 目的
- 66shogi 固有の段階的配置シーケンスを既存の将棋系バリアントを壊さずにエンジンレベルでサポートする。
- 「飛車または角のどちらか一方のみ」を機械的に強制し、ユーザーが自分でルールを守らなくても済むようにする。
- 設定項目はすべて `variants.ini` で表現できるようにし、探索ロジックに特別な分岐を埋め込まない。

### 主な設計判断
1. **汎用的なセットアップフェーズ用フック**
   - `Variant` に `setupDrops`, `setupMustDrop`, 色別の `setupDropRegion`, `setupDropPieceTypes` を追加し、一時的なドロップ専用フェーズ・有効マス・必須ポケットをバリアントごとに定義できるようにした。
   - `StateInfo` で `setupDropsActive` を追跡し、初期配置中のみドロップを強制してフェーズ終了後の通常の将棋ドロップと競合しないようにした。

2. **ピース選択グループ**
   - `Variant` に `PieceChoiceGroup` 配列を設け、`pieceChoiceGroup*` キーをパーサで解釈。各グループは対象駒集合・上限数・セットアップ限定かどうかを持つ。
   - セットアップ中の指し手生成では該当グループを参照し、例えば飛車を置いたら共有枠を消費して角のドロップを禁止する。

3. **セットアップ完了後の自動クリーンアップ**
   - 必要なドロップを終えた時点で、満たされた選択グループに属する残りのポケット駒を除去し、後続の捕獲で未使用駒が復活しないようにした。
   - 暗黙的なポケット除去でも NNUE の蓄積がずれないよう、ダーティーピース情報を拡張した。

4. **バリアント定義**
   - `src/variant.cpp` に `sixty_six_shogi_variant()` を追加し、`variants.ini` の `[66shogi:shogi]` セクションで新オプションを利用。6×6 盤、ポケット付き初期 FEN、先手段 1・後手段 6 のセットアップ領域、`pieceChoiceGroup = options=rb limit=1 required=true` などを設定している。

### 変更ファイル
- `src/variant.h / src/variant.cpp`: 新しい構造体メンバー、ヘルパー、66shogi バリアント実装。
- `src/parser.h / src/parser.cpp`: セットアップ系フィールドとピース選択グループのパース処理。
- `src/position.h / src/position.cpp`: セットアップドロップの強制、グループ使用状況の管理、自動クリーンアップ。
- `src/variants.ini`: 追加オプションを使った 66shogi ルールの宣言。

### 結果
エンジンは 66shogi を開始から終局まで処理でき、プレイヤーは自陣段への強制ドロップを交互に行い、飛車か角を必ず一方だけ配置し、6 駒の配置完了後に通常の将棋フェーズへシームレスに移行できる。構築した仕組みは、今後の配置型バリアントにも流用可能。
