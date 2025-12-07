# UniMerge: Autonomous File Orchestrator 仕様書 (v0.1)

- **Code Name**: UniMerge (Universal Merge Engine)
- **Executable**: unimerge.exe
- **Version**: 0.1 (Initial Draft)
- **Target Platform**: Windows 10/11, Windows Server 2019+ (64-bit)
- **Default Language**: 日本語 (英語切替可能 / i18n対応)

---

## 1. プロジェクト概要

### 1.1 概念：Unifying Data Intelligence

UniMerge は、Windows向けの高性能ファイル操作ツールである。
大容量データの統合、移行、同期を高速に実行し、重複ファイルの検出・整理を支援する。

**主な特徴：**

- 高速なファイルコピー・移動・マージ処理
- ハッシュ比較による重複検出（名前やタイムスタンプが異なっていても検出可能）
- 画像ファイルのEXIF情報を活用した同一写真の検出
- セキュリティ保護機能（ランサムウェア検知、操作のロールバック）

### 1.2 コアバリュー

#### Adaptive I/O Optimization（適応的I/O最適化）

実行環境のストレージ特性（レイテンシ/IOPS）を監視し、スレッド数やバッファサイズを動的に調整する。
ヒューリスティクスベースの適応アルゴリズムにより、SSD/HDD/NAS等の異なるストレージに対して最適なパフォーマンスを実現。

#### Zero-Trust Integrity（完全性検証）

- コピー処理中にファイルのエントロピー（乱雑さ）をリアルタイム監視
- ランサムウェアによる暗号化の兆候を検知した場合、処理を一時停止しユーザーに警告
- VSSスナップショットが利用可能な場合は復旧オプションを提示

#### Intelligent Deduplication（インテリジェント重複排除）

- バイナリ一致による従来型の重複検出
- EXIF情報を活用した画像/写真の同一性判定
- Perceptual Hash (pHash) による視覚的類似画像の検出

---

## 2. システムアーキテクチャ

### 2.1 I/O エンジン: FluxCore

#### アーキテクチャ

C++20 Coroutines と Windows IOCP (I/O Completion Port) を融合したノンブロッキング・ステートマシン。

#### 主要機能

| 機能 | 説明 |
|------|------|
| Zero-Copy Pipeline | FILE_FLAG_NO_BUFFERING によるカーネルバッファのバイパス。セクタアライメントされたメモリで直接I/O |
| Adaptive Throttling | システム負荷（CPU/Memory/Disk Queue）監視。閾値超過時にI/O発行を調整し、OS応答性を維持 |
| DirectStorage対応 | 将来拡張として検討（現時点では対応ハードウェアが限定的） |

### 2.2 Adaptive Tuner（適応チューナー）

| 項目 | 説明 |
|------|------|
| 機能 | 実行開始時のベンチマークと、処理中のスループット監視 |
| 最適化 | ストレージ特性（Queue Depthとレイテンシの関係等）を分析し、並列度を動的制御 |
| モード | Auto (自動調整) / Manual (ユーザー指定) / Profile (プリセット使用) |

---

## 3. UI/UX 仕様

### 3.1 国際化対応 (i18n)

| 項目 | 仕様 |
|------|------|
| デフォルト言語 | 日本語 |
| 対応言語 | 日本語、英語 |
| 実装方式 | Windows リソースファイル (.resw) + JSON fallback |
| 切替方法 | 設定画面からの言語選択、システムロケール自動検出 |

### 3.2 メイン画面構成 (Fluent Design)

```
┌──────────────────────────────────────────────────────────────────────┐
│  UniMerge  │  ダッシュボード  │  分析  │  設定  │         [v0.1.0]  │
├──────────────────────────────────────────────────────────────────────┤
│ [プロファイル: 標準コピー ▼]        [自動最適化: ON]                 │
├──────────────────────────────────────────────────────────────────────┤
│ コピー元  [ D:\Work\__________________________ ]  [履歴 ▼]          │
│ コピー先  [ E:\Backup\________________________ ]  [圧縮: なし ▼]    │
├──────────────────────────────────────────────────────────────────────┤
│ [エンジン状態]                                                       │
│ モード:   [ ミラーリング同期 ▼ ]                                     │
│ 最適化:   [ 自動調整 ]  現在の戦略: [ SSD高速モード ]                │
├──────────────────────────────────────────────────────────────────────┤
│ [オプション]                                                         │
│ [x] ランサムウェア監視        [x] 重複スキャン (EXIF + ハッシュ)     │
│ [x] 整合性検証 (xxHash)       [ ] AES-256-GCM 暗号化                 │
├──────────────────────────────────────────────────────────────────────┤
│ [進捗]                                                               │
│  転送速度:  [████████████████████████████░░░░░░░░] 1.8 GB/s          │
│  最適化状態: 安定                                                    │
├──────────────────────────────────────────────────────────────────────┤
│ [ シミュレーション ]    [ 構造分析 ]    [ 実行 ]                     │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.3 可視化機能

| 機能 | 説明 |
|------|------|
| ツリーマップ | ファイル構造を階層的に可視化。容量を占有しているフォルダを一目で特定 |
| ヒートマップ | ファイルサイズ・更新日時に基づく色分け表示 |
| 重複グループ | 検出された重複ファイルをグループ化して表示 |

---

## 4. 機能要件詳細

### 4.1 動作モード

| モード | 説明 |
|--------|------|
| Mirror Sync (ミラーリング同期) | Source と Dest を完全同期。差分のみ転送 |
| Merge (マージ) | 両方のディレクトリを統合。重複は選択的に処理 |
| Deduplicate (重複排除) | 重複ファイルを検出し、削除/移動/ハードリンク化を提案 |
| Forensic Copy | メタデータ、ACL、ADS (Alternate Data Streams)、作成日時を完全保持したコピー |

### 4.2 重複排除ロジック

#### 4.2.1 Binary Filter（バイナリフィルタ）

高速な段階的フィルタリングにより、大量ファイルから効率的に重複を検出する。

**処理順序：**

```
1. ファイルサイズ比較
   └─ サイズが異なる → 非重複として除外（※EXIF優先モード時は除く）

2. 画像/写真ファイル判定 (拡張子: jpg, jpeg, png, heic, raw, etc.)
   └─ 画像の場合 → EXIF優先モード (設定による)
       ├─ EXIF情報抽出 (撮影日時, カメラ機種, オリジナルファイル名, GPS座標等)
       ├─ EXIF一致 → 同一写真の可能性高 (サイズ違いでも同一と判定可能)
       └─ EXIF不一致/なし → 次のステップへ

3. 部分ハッシュ比較 (先頭64KB + 末尾64KB)
   └─ 部分ハッシュ不一致 → 非重複として除外

4. 完全ハッシュ比較 (xxHash64)
   └─ 完全一致 → 重複確定
```

**EXIF優先モード設定：**

| 設定 | 動作 |
|------|------|
| 無効 (off) | EXIF情報を無視し、バイナリハッシュのみで判定 |
| 参考 (hint) | EXIF一致を重複候補として提示（ユーザー確認） |
| 優先 (priority) | EXIF一致でサイズ違いも同一写真として自動判定 |

**抽出するEXIF情報：**

| タグ | 説明 | 重要度 |
|------|------|--------|
| DateTimeOriginal | 撮影日時 | 高 |
| Make / Model | カメラメーカー/機種 | 高 |
| ImageUniqueID | 画像固有ID | 最高 |
| DocumentName | ドキュメント名 | 中 |
| OriginalRawFileName | 元のRAWファイル名 | 高 |
| GPSLatitude / GPSLongitude | GPS座標 | 中 |
| ExifImageWidth / ExifImageHeight | 元の解像度 | 中 |

**EXIF同一性判定ロジック：**

判定は以下の優先順位で実行され、最初に合致した条件で結果が確定する。

1.  **レベル1: 完全一致 (Strong Match)**
    *   **条件:** `ImageUniqueID` が一致する。
    *   **判定:** 最も信頼性が高い一致。サイズやメタデータが異なっていても同一のオリジナル写真から派生したとみなす。

2.  **レベル2: 高信頼一致 (High-Confidence Match)**
    *   **条件:**
        *   `DateTimeOriginal` + `Make` + `Model` がすべて一致する。
        *   **かつ、** 元の解像度 (`ExifImageWidth` / `ExifImageHeight`) の差が許容範囲内（例: 面積比で2%未満）。
    *   **判定:** ほぼ同一の写真（例: メタデータ編集のみが異なる）。

3.  **レベル3: 類似候補 (Similar Candidate - Resized/Edited)**
    *   **条件:**
        *   `DateTimeOriginal` + `Make` + `Model` がすべて一致する。
        *   **しかし、** 解像度の差が許容範囲を超える。
    *   **判定:** リサイズ、トリミング、大幅な再編集が加えられた可能性のある写真。

4.  **レベル4: 位置情報による一致 (GPS Match)**
    *   **条件:**
        *   `DateTimeOriginal` が一致する。
        *   **かつ、** GPS座標 (`GPSLatitude` / `GPSLongitude`) が非常に近い（例: 誤差10m以内）。
        *   ※カメラ機種が異なる場合でも一致と判定されうる（例: 同じ場所でスマホとデジカメで同時撮影）。
    *   **判定:** 状況証拠による一致。ユーザー確認を推奨。

5.  **レベル5: RAW+JPEG連携 (RAW/JPEG Pair)**
    *   **条件:**
        *   `OriginalRawFileName` が一致し、かつ `DateTimeOriginal` が一致する。
    *   **判定:** カメラ内で同時に生成されたRAWファイルとJPEGファイルのペアである可能性が高い。

上記いずれにも一致しない場合は、EXIF情報による同一性はないと判断し、次のハッシュ比較ステップに進む。モード設定（`hint`/`priority`）に応じて、これらの判定結果の扱いは変わる。

#### 4.2.2 Semantic Filter（意味的フィルタ）- Experimental

Binary Filterに加え、より高度な類似性検出を行う。

| 対象 | 手法 | 説明 |
|------|------|------|
| 画像 | Perceptual Hash (pHash) | 視覚的に類似した画像を検出（リサイズ・圧縮・形式変換後も対応） |
| テキスト/コード | 正規化 + ハッシュ | 行末コード (CRLF/LF)、空白、BOMの違いを無視して比較 |
| ドキュメント | 将来検討 | PDF/Office文書の内容類似性判定 |

### 4.3 セキュリティ & 安全性 (Guardian)

#### Ransomware Shield（ランサムウェア監視）

| 項目 | 説明 |
|------|------|
| 検知方式 | ファイルエントロピー分析（暗号化されたファイルは高エントロピー） |
| アクション | エントロピー急上昇を検知時、書き込みを一時停止しユーザーに警告 |
| 復旧支援 | VSSスナップショットが利用可能な場合、復旧オプションを提示 |

#### Safe Undo（安全な取り消し）

| 項目 | 説明 |
|------|------|
| 記録方式 | SQLiteデータベースに全操作履歴（コピー、移動、名前変更）を記録。 |
| ロールバック | `unimerge.exe /undo [JobID]` で記録された操作の逆処理を実行し、復元を試みる。 |
| **削除の取り扱い** | **直接的なファイル削除はロールバック不可。** 代わりに「隔離」機能で安全性を確保する。 |
| **隔離 (Quarantine)** | 重複排除などで削除対象とされたファイルは、実際には隔離フォルダへ移動される。ユーザーはここから手動で復元可能。 |
| 隔離先と保持期間 | 隔離先パスと保持期間は設定可能（デフォルト: `C:\$UniMerge_Quarantine\` , 30日間）。期間超過後に自動削除される。 |

#### 暗号化転送

| 項目 | 説明 |
|------|------|
| 方式 | AES-256-GCM |
| 用途 | リモートストレージへの転送時、機密データの保護 |
| 備考 | 将来的にポスト量子暗号 (PQC) 対応を検討 |

---

## 5. CLI / Automation

```
unimerge.exe [Command] [Options]
```

### コマンド例

#### 高速コピー（自動最適化）

```powershell
unimerge.exe /copy /source:"D:\Work" /dest:"E:\Backup" /optimize:auto
```

#### ミラーリング同期

```powershell
unimerge.exe /sync /source:"D:\Data" /dest:"\\NAS\Backup" /delete-extra
```

#### 重複スキャン（EXIF優先モード）

```powershell
unimerge.exe /scan:duplicates /target:"D:\Photos" /exif-mode:priority /report:"C:\dupes.json"
```

#### セマンティックスキャン（類似画像検索）

```powershell
unimerge.exe /scan:semantic /target:"D:\Photos" /report:"C:\similar.json"
```

#### 緊急停止とロールバック

```powershell
unimerge.exe /emergency-stop
unimerge.exe /undo --last-job
```

### CLI オプション一覧

| オプション | 説明 |
|------------|------|
| /source:"path" | コピー元パス |
| /dest:"path" | コピー先パス |
| /target:"path" | スキャン対象パス |
| /optimize:auto\|manual\|profile | 最適化モード |
| /exif-mode:off\|hint\|priority | EXIF判定モード |
| /report:"path" | レポート出力先 (JSON/CSV) |
| /dry-run | シミュレーション実行（実際には変更しない） |
| /lang:ja\|en | 出力言語指定 |

### 終了コード (Exit Codes)

自動化スクリプトから実行結果を判定できるよう、以下の終了コードを定義する。

| コード | 意味 | 説明 |
|:-------|:-----|:-----|
| 0 | 成功 (Success) | 全ての操作が正常に完了した。 |
| 1 | 警告 (Warning) | 操作は完了したが、重複が検出された、あるいは同期差分がなかったなど、特記事項がある。詳細はログを確認。 |
| 2 | パラメータエラー (Parameter Error) | コマンドライン引数が不正。 |
| 3 | I/Oエラー (I/O Error) | ファイルやディレクトリへのアクセス権限がない、パスが存在しない、ディスク空き容量不足など。 |
| 4 | ユーザーによる中断 (User Canceled) | ユーザーによって処理がキャンセルされた。 |
| 5 | 安全機能による停止 (Safety Stop) | ランサムウェア検知など、Guardian機能によって処理が緊急停止された。 |
| 9 | 不明なエラー (Unknown Error) | 上記以外の予期せぬエラーが発生した。 |

---

## 6. 内部実装仕様

### 6.1 コア・データ構造

```cpp
// UniMerge File Node
struct MergeNode {
    std::wstring path;
    uint64_t size;
    uint64_t hashPartial;   // 部分ハッシュ (先頭+末尾)
    uint64_t hashFull;      // 完全ハッシュ (xxHash64)

    // EXIF情報 (画像ファイルのみ)
    struct ExifData {
        std::wstring dateTimeOriginal;
        std::wstring cameraMake;
        std::wstring cameraModel;
        std::wstring imageUniqueId;
        std::wstring originalRawFileName;
        std::optional<double> gpsLatitude;
        std::optional<double> gpsLongitude;
        uint32_t originalWidth;
        uint32_t originalHeight;
        bool hasValidExif;
    } exif;

    // 意味的特徴量 (Semantic Filter用)
    std::vector<uint8_t> perceptualHash;  // pHash (64bit or 256bit)

    // 状態管理
    struct State {
        bool isLocked;
        bool isCompressed;
        float entropyScore;  // 0.0 - 1.0 (ランサムウェア検知用)
    } state;
};

// Adaptive Tuning Context
struct TuningContext {
    double currentLatencyMs;
    double throughputMBps;
    double throughputMA;       // 移動平均
    int activeThreadCount;
    int recommendedThreadCount;
    size_t currentBufferSize;
    size_t optimalBufferSize;

    void Update(double latencySample, double throughputSample);
    void AdjustParameters();
};

// EXIF比較結果
enum class ExifMatchResult {
    NoExif,           // EXIF情報なし
    NotMatch,         // 不一致
    PartialMatch,     // 部分一致（要確認）
    FullMatch         // 完全一致（同一写真）
};
```

### 6.2 開発スタックとライブラリ

| カテゴリ | 技術/ライブラリ | バージョン | 備考 |
|----------|----------------|-----------|------|
| **言語** | C++20 | MSVC 2022+ | Coroutines, Ranges, Concepts, std::format |
| **ビルド** | CMake | 3.25+ | |
| **GUI** | WinUI 3 | Windows App SDK 1.4+ | Fluent Design, XAML |
| **ハッシュ** | xxHash | 0.8.2+ | 高速非暗号化ハッシュ |
| **暗号化ハッシュ** | BLAKE3 | 1.5+ | 暗号学的用途・整合性検証 |
| **EXIF** | libexif / exiv2 | - | EXIF/IPTC/XMP読み取り |
| **画像処理** | stb_image | - | 画像デコード (pHash計算用) |
| **pHash** | pHash / OpenCV | - | Perceptual Hash計算 |
| **圧縮** | Zstd | 1.5+ | リアルタイム圧縮転送 |
| **データベース** | SQLite | 3.40+ | 操作履歴・設定保存 |
| **i18n** | Windows .resw | - | リソースファイルによるローカライズ |

### 6.3 ビルド要件

| 要件 | 詳細 |
|------|------|
| OS | Windows 10 version 1903+ / Windows 11 |
| コンパイラ | MSVC 19.30+ (Visual Studio 2022) |
| Windows SDK | 10.0.19041.0+ |
| Windows App SDK | 1.4+ |

---

## 7. ロードマップ

| Phase | 名称 | 内容 |
|-------|------|------|
| Phase 1 | **Core** | IOCPエンジンの完成、基本コピー/同期機能、CLI実装 |
| Phase 2 | **Optimize** | Adaptive Tuner実装、パフォーマンス最適化 |
| Phase 3 | **Detect** | EXIF重複検出、Binary Filter完成、Ransomware Shield |
| Phase 4 | **Semantic** | pHash実装、Semantic Filter (Experimental) |
| Phase 5 | **Cloud** | S3/Azure Blob へのダイレクトストリーミング対応（検討中） |

---

## 8. 検討事項・将来対応

以下は現時点では技術的制約や優先度の観点から実装を見送るが、将来的に対応を検討する項目：

| 項目 | 理由 | 代替/対応方針 |
|------|------|--------------|
| GPU DirectStorage | 対応ハードウェア（NVMe + 対応GPU）が限定的 | 将来的にハードウェア普及後に対応 |
| USNジャーナル完全コピー | ボリューム単位管理のため、ファイル単位コピーは困難 | 別途USNジャーナルエクスポート機能として検討 |
| ポスト量子暗号 (PQC) | 標準化進行中、ライブラリ成熟待ち | AES-256-GCMを使用し、PQC標準化後に対応 |
| ONNX Runtimeによる高度なAI推論 | オーバーヘッド懸念 | セマンティック検索のオプション機能として検討 |
| クロスプラットフォーム対応 | Windows専用機能（IOCP, WinUI等）に依存 | Windows専用として開発継続 |
