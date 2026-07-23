# 障害報告書 #001：Webサービス応答停止

## サマリー
監視対象のWebサービス（Nginx）が停止し、外形監視により検知。
アラートが設計どおり発報し、手動復旧により回復した。
本件は監視基盤の動作検証を目的とした計画的な障害試験である。

## 基本情報
| 項目 | 内容 |
|---|---|
| インシデントID | INC-001 |
| 重要度 | Critical |
| 検知方法 | Prometheus 外形監視（Blackbox Exporter） |
| 発報アラート | WebServiceDown |
| ステータス | 復旧済み |

## タイムライン
| 時刻 | 出来事 |
|---|---|
| 06:02 | Nginx コンテナを停止（障害発生） |
| 06:02 | probe_success が 1 → 0 に変化（監視が異常を検知） |
| 06:03 | WebServiceDown アラートが PENDING → FIRING に遷移、Alertmanager へ通知 |
| 06:05 | Nginx コンテナを再起動（復旧作業） |
| 06:05| probe_success が 0 → 1 に回復、アラートが Inactive へ戻る |

## 影響
- 影響範囲：Webサービスにアクセスするすべてのユーザー
- 影響内容：サイトへのアクセス不可（HTTP応答なし）
- 継続時間：約3分

## 根本原因
Nginx コンテナのプロセス停止により、HTTP リクエストへの応答が不能になった。
（今回は検証目的で意図的に `docker compose stop nginx` を実行した）

## 対応内容
1. Prometheus の Alerts 画面で FIRING を確認
2. Alertmanager への通知到達を確認
3. `docker compose start nginx` によりサービスを再起動
4. probe_success の回復およびアラートの解消を確認

## 検知が機能した点（良かった点）
- 外形監視により、サービス停止を15秒以内に数値として検知できた
- `for: 1m` の設定により、瞬断ではなく継続的な障害としてのみ発報する設計が機能した
- アラートが Prometheus から Alertmanager まで正しく連携した

## 再発防止・改善案
- Nginx コンテナに `restart: unless-stopped` を設定済みのため、プロセス異常終了時は自動復旧する
- 今後の改善：Alertmanager から Slack への通知連携を追加し、検知から通知までを自動化する
- 今後の改善：コンテナの自動復旧が効かないケース（設定ミス等）を想定した監視の追加を検討する