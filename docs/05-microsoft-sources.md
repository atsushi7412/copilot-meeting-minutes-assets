# Microsoft仕様確認

確認日：2026年10月21日。以下はMicrosoft公式資料に基づく実装上の根拠。テナント固有の設定と実動作は別途確認する。

- [Agent Builderでの作成](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/agent-builder-build-agents)：構成画面の名前・説明・指示・ナレッジ・候補プロンプトとTry itの手順。指示は8,000文字以内。Agent Builderだけで外部サービスのアクションを作成する方式ではない。
- [知識源の追加](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/agent-builder-add-knowledge)：SharePoint、OneDrive、Teams、Outlookメール等の設定。準備中の資料は回答で使われない。指定ソースを優先する設定は、一般知識を完全遮断する保証ではない。
- [宣言型エージェントの知識源](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/knowledge-sources)：会議情報と書き起こしへの参照、知識源ごとのライセンス条件。メール知識と予定表イベント取得を同一機能と扱わない。
- [カスタムスキルの追加](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/agent-builder-add-skills)：Frontier限定プレビュー。ZIP直下のSKILL.mdと任意リソースで構成する。スキル指示は20,000文字未満。IB利用テナントでは不可。
- [カスタムスキルの概要](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/declarative-agent-skills)：対応形式と制約。現在のプレビューではSkillsと埋め込みファイルの併用は非対応。
- [エージェントの共有と管理](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/agent-builder-share-manage-agents)：組織内の共有、編集者と利用者の違い、知識源の権限確認。
- [Copilotの製品区分](https://learn.microsoft.com/en-us/microsoft-365/copilot/copilot-overview)：職場向けMicrosoft 365 Copilotの利用区分の確認先。
- [イベントトリガー](https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-triggers-about)：イベントに応じた実行と課金・接続条件。無操作の自動生成を追加する場合に参照する。

## 実機で確かめる事項

正式な割当ライセンス名、エージェント作成・共有の可否、Teams会議書き起こしの取得範囲と反映時間、対象回のOutlook会議設定の取得項目、SSOTの対応形式と更新反映、カスタムスキルの表示と動作。原本を作成しただけでは、これらを確認済みにしない。
