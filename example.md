# ADR 0001: ApacheとTomcatを使用したWebアプリのKubernetes構成

## 状態
承認済み（2025-04-10）

## 背景
当社のJavaウェブアプリケーションをKubernetes環境に移行するにあたり、ApacheとTomcatの連携方法、Pod構成、およびログ管理の方針を決定する必要がある。特に高可用性、スケーラビリティ、および効率的なログ管理が要件となっている。

## 決定内容と理由
![](./img/example_2025-03-28%20071822.png)

1. **マルチコンテナPod構成**: 各Podに Apache HTTP Server と Tomcat を配置する
    - Apache HTTP ServerがリバースプロキシとしてTomcatへのリクエストを転送する構成が最適
    - mod_jkを使用してApacheとTomcatの接続を確立（AJP接続）
    - 単一のPod内での通信により、レイテンシを最小限に抑える
    - セキュリティレイヤーとしてApacheが機能し、Tomcatへの直接アクセスを防止

2. **PersistentVolume (PV)によるログ管理**: アプリケーションログとアクセスログを永続ボリュームに出力する
    - Pod再起動時もログが永続化され、トラブルシューティングが容易になる
    - ReadWriteMany (RWX) アクセスモードを使用することで複数Podからの書き込みが可能
    - NFS StorageClassを利用して、クラスタ全体でログの一元管理を実現

3. **Sidecarパターン**: ログ収集用のコンテナをサイドカーとして実装する
    - Fluentdコンテナをサイドカーとして配置し、ログの収集と転送を担当
    - メインアプリケーションコンテナの責務を分離しクリーンなアーキテクチャを維持
    - ログ収集の独立したスケーリングが可能

## 影響
- Kubernetes manifestsの作成が必要（Deployment, Service, PVC等）
- CI/CDパイプラインの更新
- 以下の設定が必要:
  - Apacheの設定（mod_jk、バーチャルホスト、ロードバランシング）
  - Tomcatの設定（AJPコネクタ、セッション管理）
  - PVClaimの設定とNFSサーバーのプロビジョニング
  - ログローテーション設定

## 比較・検討内容
1. **Pod構成の検討**:
   - 採用案: マルチコンテナPod（Apache + Tomcat）
     - 理由: レイテンシ最小化、シンプルな管理、セキュリティ強化
   - 不採用案1: シングルコンテナPod（Tomcatのみ）
     - 理由: セキュリティ機能の制限、静的コンテンツ配信の非効率性
   - 不採用案2: 別々のPod（Apache Pod + Tomcat Pod）
     - 理由: ネットワークオーバーヘッド、運用コスト増加

2. **ログ管理方式の検討**:
   - 採用案: PersistentVolume + Fluentdサイドカー
     - 理由: ログの永続化、柔軟なログ収集、Kubernetes標準準拠
   - 不採用案1: ホストボリューム
     - 理由: 環境依存性、ノード間ログ共有の困難さ
   - 不採用案2: ログフォワーダーサービス
     - 理由: 追加インフラコスト、運用の複雑化

3. **スケーリング戦略の検討**:
   - 採用案: 水平スケーリング（Pod単位）
     - 理由: 柔軟なスケーリング、高可用性の確保
   - 不採用案: 垂直スケーリング（リソース増強）
     - 理由: 物理的な制限、コスト効率の悪さ

4. **ログ収集方式の検討**:
   - 採用案: Fluentdサイドカー
     - 理由: ログ収集の独立性、柔軟な設定
   - 不採用案1: DaemonSet方式
     - 理由: ノードレベルの障害影響
   - 不採用案2: アプリケーション内での直接ログ転送
     - 理由: アプリケーションの責務増加、保守性の低下

5. **セッション管理の検討**:
   - 採用案: Redisによるセッション共有
     - 理由: スケーラビリティ、高可用性
   - 不採用案1: スティッキーセッション
     - 理由: 負荷分散の非効率性
   - 不採用案2: データベースセッション
     - 理由: パフォーマンスオーバーヘッド

## 参考資料
- [社内設計レビュー会議（2025-04-05）](/project-documentation_for-ai/meeting-transcripts/2025-04/2025-04-05-技術設計レビュー会議.md)