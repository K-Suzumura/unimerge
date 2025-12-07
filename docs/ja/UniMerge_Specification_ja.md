UniMerge: Autonomous File Orchestrator 仕様書 (v3.0)Code Name: UniMerge (Universal Merge Engine)Executable: unimerge.exeVersion: 3.0 (Next-Gen Architecture)Target Platform: Windows 10/11, Windows Server 2019+ (64bit / AVX2 Required)Dev Stack: C++23 (MSVC), ONNX Runtime (AI Edge Computing)1. プロジェクト概要1.1 概念：Unifying Data IntelligenceUniMerge は、従来の「ファイル操作ツール」の枠を超越した、自律型ファイル・オーケストレーション・エンジンである。OSのファイルシステムとユーザーの間に介入し、ペタバイト級のデータ統合、移行、同期を「理論限界速度」で実行するだけでなく、データの意味理解やセキュリティ保護までを自律的に遂行する。1.2 コアバリュー (The Trinity Core)Neural I/O Optimization (AI駆動の極限速度):従来のルールベース制御を廃止。内蔵された軽量AIモデルが、実行環境のストレージ応答（Latency/IOPS）をミリ秒単位で監視・学習し、スレッド数やバッファサイズを動的に「自己進化」させる。Zero-Trust Integrity (完全な安全性):コピー処理中にファイルエントロピーをリアルタイム監視。ランサムウェアによる「意図しない暗号化（破壊）」を検知した瞬間、プロセスを凍結しVSSスナップショットからの即時復旧を提案する。Semantic Deduplication (意味的重複排除):バイナリ一致だけでなく、「内容は同じだが保存形式が異なるデータ」（例: BMPとPNG、リサイズされた画像、行末コードが違うテキスト）をAIが同一視し、統合を提案する。2. システムアーキテクチャ2.1 I/O エンジン: "FluxCore" (IOCP + Coroutines)Architecture: C++20 Coroutines と IOCP を融合したノンブロッキング・ステートマシン。Zero-Copy Pipeline: カーネルメモリからユーザーメモリへのコピーを排除する FILE_FLAG_NO_BUFFERING と、GPU Direct Storage (将来拡張) を見据えたメモリアライメント設計。Adaptive Throttling: システム負荷（CPU/Memory/Disk Queue）が閾値を超えた場合、マイクロ秒単位でI/O発行を調整し、OSのフリーズ（プチフリ）を完全回避する。2.2 Neural Tuner (自己調整モジュール)機能: 実行開始直後のベンチマークだけでなく、処理中も常にスループットを監視。学習: 「このSSDはQueue Depth 32を超えるとレイテンシが悪化する」といった特性を学習し、並列度を自律制御する。3. UI/UX 仕様3.1 メイン画面構成 (Modern Fluent Design)+----------------------------------------------------------------------------------+
|  UniMerge  |  Dashboard  |  Analytics  |  Settings  |            [v3.0.0]      |
+----------------------------------------------------------------------------------+
| [Profile: Enterprise Migration ▼]   [Auto-Pilot: ON (AI Optimizing)]             |
+----------------------------------------------------------------------------------+
| Source [ \\?\Volume{GUID}\Data _________________________ ] [Snapshot: Latest]    |
| Dest   [ \\?\192.168.1.10\Backup _______________________ ] [Compression: Zstd]   |
+----------------------------------------------------------------------------------+
| [FluxCore Status]                                                                |
| Mode:   [ Autonomous Sync (自律同期) ▼ ]                                         |
| Tuning: [ Neural Adaptive (AI制御) ]  Current Strategy: [ NVMe Saturation ]      |
+----------------------------------------------------------------------------------+
| [Intelligence Layer]                                                             |
| [x] Ransomware Shield (エントロピー監視)    [x] Smart Semantic Scan (類似画像等) |
| [x] Integrity Verify (XXHASH64)             [ ] Quantum-Safe Encrypt (AES-256)   |
+----------------------------------------------------------------------------------+
| [Visualization]                                                                  |
|  I/O Stream:  [||||||||||||||||||||||||||||||||||||||||||||] 2.4 GB/s            |
|  AI Confidence: 98% (Optimization Stable)                                        |
+----------------------------------------------------------------------------------+
| [ Simulation (Dry Run) ]    [ Analyze Topology ]    [ EXECUTE (Engage) ]         |
+----------------------------------------------------------------------------------+
3.2 可視化機能Topological Map: ファイル構造を3Dまたはツリーマップで可視化し、容量を食っているフォルダを一目で特定。Heatmap: ディスク上の物理的な断片化状況や、アクセス頻度をヒートマップ表示（デフラグ提案機能）。4. 機能要件詳細4.1 動作モードAutonomous Sync: AIが Source/Dest の差異を分析し、最適な手順（移動・コピー・ハードリンク）を自動生成。Semantic Merge: 類似ファイルを検出し、「高画質版を残して低画質版を削除」「重複ドキュメントを最新版に統合」などを提案。Forensic Copy: メタデータ、ACL、ADS、作成日時だけでなく、USNジャーナル情報も含めた完全なフォレンジックコピー。4.2 重複排除ロジック (Semantic & Binary)Binary Filter: サイズ → 部分ハッシュ → 完全ハッシュ（従来型）。Semantic Filter (Experimental):Image: Perceptual Hash (pHash) を用いて「見た目が同じ」画像を検出。Text/Code: トークン化による類似度判定で「バージョン違い」を検出。4.3 セキュリティ & 安全性 (Guardian)Ransomware Shield: 書き込み対象ファイルのエントロピー（乱雑さ）が急激に上昇した場合（暗号化の兆候）、書き込みを即時遮断しユーザーに警告。Safe Undo: 操作履歴をSQLiteデータベースに記録。誤操作時は unimerge.exe /undo [JobID] で逆操作を生成し、元の状態へロールバック可能。5. CLI / Automation (DevOps Ready)unimerge.exe [Command] [Options]Command Examples:自律バックアップ:unimerge.exe /autopilot /source:"D:\Work" /dest:"Z:\Backup"AIがモードを自動判定して実行。セマンティックスキャン:unimerge.exe /scan:semantic /target:"D:\Photos" /report:"C:\dupes.json"類似画像を検索してJSONレポートを出力。緊急停止とロールバック:unimerge.exe /emergency-stopunimerge.exe /undo --last-job6. 内部実装仕様 (Technical Deep Dive)6.1 コア・データ構造// UniMerge File Node
struct MergeNode {
    std::wstring path;
    uint64_t size;
    uint64_t signature; // XXHASH64

    // 意味的特徴量 (AI用)
    std::vector<float> semanticVector;

    // 状態管理
    struct State {
        bool isLocked;
        bool isCompressed;
        bool isEncrypted;
        float entropyScore; // 0.0 - 1.0
    } state;
};

// AI Tuning Context
struct NeuralContext {
    double currentLatency;
    double throughputMA; // 移動平均
    int recommendedThreadCount;
    size_t optimalBufferSize;

    void Update(double latencySample); // 強化学習Step
};
6.2 開発スタックとライブラリCore: C++23 (Modules, Coroutines)AI/ML: ONNX Runtime (軽量モデルによる推論)Hashing: xxHash (v0.8.2), BLAKE3 (暗号学的ハッシュが必要な場合)Compression: Zstd (リアルタイム圧縮転送用)GUI: WinUI 3 または Qt 6 (GPUアクセラレーション必須)7. ロードマップ (Evolution Path)Phase 1 (Flux): IOCPエンジンの完成と基本コピー機能。Phase 2 (Neural): AIチューナーの実装と最適化。Phase 3 (Semantic): 意味的重複排除と画像認識の実装。Phase 4 (Cloud): S3/Azure Blob へのダイレクトストリーミング対応。
