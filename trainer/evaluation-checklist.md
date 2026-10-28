# 採点運用・合否関数（講師用・架空演習）

本書は原子採点基準（trainer/scoring-ledger.json）と集計器（tools/score.mjs）の運用を説明する。全件架空。採点器は人が付けた項目判定を集計するものであり、議事録内容の意味を自動で正しく判定したとは主張しない。

## 1. 採点の流れ
1. 受講者が各ケースの入力束でCopilotに議事録初稿を作らせる（実機・別工程・本v0.2では未実施）。
2. 講師が scoring-ledger.json の各項目を人判定し、itemIdごとに PASS / FAIL（適用項目へのNA申告はINVALID） を run-records（mmca-runrecords/1）へ記録する。
3. Codex が `node tools/score.mjs trainer/scoring-ledger.json <run-records>.json` を実行し、dimension別・run別の集計を得る。
4. 合否は人が確認する。集計器は exit 0 固定で、結果を「合格」と主張しない。

## 2. dimension と原子化
- dimension は content / entity / gate の3つ。混ぜて1つの平均にしない（旧 M-03 の平坦な「27」を廃止）。
- アクションは 作業 / 担当 / 期限 / 根拠 の4原子に分割して計数する。複合記述は原子へ分割後に数える（C01に加えR3でC08の資料整理アクションも4原子へ分解）。
- 会議設定の欠落項目（主催者・招待者・予定時刻・会議URL）も各々独立項目で計数する（R3でC07-C-02を4項目へ分解）。
- 分母は入力とmode/purposeから先に固定する。出力欠落で減らさない。重複したC01-C-16/C03-C-03は除去し、診断CONTAMに初稿本文や人物表記を要求しない。各生成ケースの重大誤りgateはcontent加点と独立。
  - content 合計62（C01=18, C02=5, C03=6, C04=2, C05=5, C06=3, C07=6, C08=4, IMP-before=3, IMP-after=3, CONTAM=0, BIZ-CONFLICT=3, URL-IDMISS=2, URL-TOKFRAG=2）。R1是正でC03にアクション（改訂案共有）の作業/担当/期限/根拠4原子（C03-C-03-TASK/OWNER/DUE/EVIDENCE）を追加しC03=2→6、content総分母は58→62。
  - entity 合計21（C01=7, C02=0, C03=1, C04=0, C05=1, C06=4, C07=2, C08=1, IMP-before=2, IMP-after=2, CONTAM=0, BIZ-CONFLICT=1, URL-IDMISS=0, URL-TOKFRAG=0）
  - gate 合計32（C01=4, C02=1, C03=1, C04=6, C05=1, C06=1, C07=1, C08=4, IMP-before=2, IMP-after=2, CONTAM=1, BIZ-CONFLICT=2, URL-IDMISS=4, URL-TOKFRAG=2）
  - C01は29原子。採点項目の正本はscoring-ledger.json。CONTAMは混入検出gateだけを採点する。

## 3. 合否関数（dimension別・run別。合算で救済しない）
- content：合格＝満たした項目数 / 適用content項目数 ≥ 0.95。適用集合（分母）は case/variant（入力）だけで確定し、固定された適用項目数で数える（判定欠落時も分母を保つ）。run-record が適用項目に NA を自己申告して分母から外すことは許さない（その run は INVALID）。分母が0になるのは入力由来で適用項目が1件も無い dimension のときだけで、その場合のみ N/A（0/0を100%にしない）。
- entity：誤り0（全件一致必須）。網羅率で相殺しない。適用0件は N/A。
- gate：全件PASS必須。1件でもFAILなら当該runを不合格とし、content網羅率と独立に判定。適用0件は N/A。R3是正（統合指摘#8）：各生成ケース（purpose=generation。CONTAMは診断専用のため対象外）に `<CASE>-G-MAJOR`（担当・期限・決定・金額・根拠等の重大事実について根拠のない創作/誤記が0件であることを求めるgate）を設ける。content が原子を増やして分母を広げた結果 19/20=95% のように閾値を満たしても、重大事実の創作/誤記が1件でもあれば CASE-G-MAJOR が独立にFAILし、content側の網羅率でgateを相殺しない。CASE-G-MAJORはcontent加点へ混ぜない。
- run合格＝予定した各runが個別に content≥95%（またはN/A）・entity全合格・gate全合格。複数runを合算して不合格runを救済しない。
- 標準版とスキル版の実績を混ぜない（variantで分離）。
- mode（evaluation/business）・purpose（generation/diagnostic）もrunを分離する軸。purpose=diagnosticのrun（CONTAM）は、混入検出項目（requiresState=contamination-detected）がPASSならdiagnostic.verdict=PASSだが、同時にgeneration.status=INCOMPLETE・generationEligible=false・cleanGenerationSuccess=falseとなり、清浄生成成功（cleanGenerationSuccesses集計）には数えない。diagnosticPasses集計とcleanGenerationSuccesses集計を絶対に合算・混同しない。item.modeScopeとrun.runModeが不整合な項目は当該runに非適用（評価モードの汚染検出項目を業務runへ混ぜない）。

## 4. run無効化（NOT PASS・閉集合）
scoring config の runInvalidation に対応。いずれかに該当する run は COMPLETE にしない：
- 予定run欠落（planned-run-missing → overall INCOMPLETE）
- 未知run（plannedRunsに無い runId → INVALID）
- 重複項目（同一itemIdの多重 → INVALID duplicate-item）
- 未実施（適用項目の判定欠落 → INCOMPLETE not-executed）
- inputFingerprint / sourceFingerprint の不一致（→ INVALID）
- case / variant / inputVersion の欠落・null・誤型・不一致（必須かつ予定runと厳密一致。欠落/nullは missing-field、型違いは type-mismatch、値違いは case-mismatch / variant-mixed / version-mismatch → いずれも INVALID）
- variant混在（run.variant ≠ planned.variant → INVALID variant-mixed）
- 不正な判定ラベル（適用項目の result が PASS|FAIL 以外。NA の自己申告や BANANA 等の未知ラベル・null・誤型 → INVALID invalid-result）
- 項目マッピング不能（適用集合外のitemId → INVALID item-unmappable）
- 1項目多重マッチ（→ INVALID duplicate-item）
- repeatIndex／runMode／runPurpose の欠落・型違い・予定runとの不一致（→ INVALID）
- 必須マトリクス不一致（config.scoring.requiredRunMatrixを決定論展開した{case,variant,repeatIndex,mode,purpose}集合とplannedRunsが完全一致しない → 台帳不備INVALID、または必須run未実施INCOMPLETE）

## 4b. SSOT本文の提供経路（統合指摘#5・obs#5是正）
- 採点・審査のため人や上院に提示する際は、plannedRun の `inputFiles` と `sourceFiles` に挙げたファイルの**全内容（本文そのもの）**を実際に提供する。ファイル名・パスだけを示して「提供した」とみなしてはならない。
- 既に組成済みの束（例：input-bundle.txt が参照SSOTの必要箇所を本文として内包している場合）がその内容を包含していれば、同じ内容の二重添付は不要（重複提供を強制しない）。
- 新設した BIZ-CONFLICT と URL-IDMISS/URL-TOKFRAG は、`training/biz-conflict/SSOT-a.txt` ・ `SSOT-b.txt` ・ `training/SSOT-demo.txt` の本文を input-bundle.txt 側に内包していない（パス参照のみ）。したがって審査・採点時は、当該 sourceFiles の本文を**別途実際に添付**する。欠けた source の内容を推測で埋めてはならない。
- `docs/02-ssot-and-improvement.md` 等で「人が査読済み」と記す段落は、AI上院（Codex／Opus統括配下の独立技術審査）による構造・整合性の検査を指す場合がある。人による意味査読（本番相当の人の目での最終校正）とは区別し、取り違えないように明記する（本v0.2で人による意味査読は未実施）。

## 5. fingerprint束縛（obs#7）
- 採点の対象（どの ledger／どの入力束を正準とするか）を取り違えない：
  - 実際の Copilot 採点では、生成した candidate の `payload/instructor/trainer/scoring-ledger.json` を ledger として使い、Fingerprint の対象は**同じ candidate の training 内にある最終入力束（compiled 済みの束）**にする。source 側の生入力束を compiled の束と取り違えない。
  - source 側 ledger（source/trainer/scoring-ledger.json）と calibration-fixtures を使うのは §7 の合成集計試験であり、Copilot 実機採点とは別物として区別する。
- plannedRun の inputFingerprint / sourceFingerprint は null。score.mjs が inputFiles / sourceFiles の実バイトから正準値を算出し、それを真値とする。
- run-records の自己申告 fingerprint は、この file 由来の正準値と照合する。run の存在確認だけ、または自己申告どうしの一致では合格にしない。
- ledger の期待fingerprintを後で固定する場合も、score.mjs は「期待値が実ファイルの正準値と一致するか」を検査する（期待値の実体束縛の担保）。
- 実ハッシュ値は本教材作成時には書いていない（シェル非実行のため。捏造しない）。Codex が次で算出する：
  - `node tools/score.mjs --print-fingerprints trainer/scoring-ledger.json`
  - 出力の inputFingerprint / sourceFingerprint を必要に応じて ledger.plannedRuns へ固定できる（固定後も score.mjs が実バイトと再照合する）。

## 6. 3区別（§6）
- 集計試験：合成ラベル（calibration-fixtures/*.json）をローカルで集計し、dimension別網羅率・run別合否を再現一致で確認する。議事録本文は入力に取らない。
- 採点校正：項目・試料・記録欄の対応検査のみ（人による校正は未実施）。
- Copilot行動試験：別ID・実機未実施。
採点器は人判定の集計であり、意味の自動判定ではない。

## 7. 集計試験の手順（ローカル・合成）
```
node tools/score.mjs trainer/scoring-ledger.json trainer/calibration-fixtures/fixture-01-dimensions.json
node tools/score.mjs trainer/scoring-ledger.json trainer/calibration-fixtures/fixture-02-invalidation.json
node tools/score.mjs trainer/scoring-ledger.json trainer/calibration-fixtures/fixture-03-atomic-boundary.json
node tools/score.mjs trainer/scoring-ledger.json trainer/calibration-fixtures/fixture-04-mode-and-gate-labels.json
```
各 fixture の `_doc.expected` が、集計器の出力（perRun の verdict・overall）と一致することを人が確認する。
- fixture-01：content<95%のFAIL、entity/gateのN/A、gate独立FAIL（C08はR3でcontent4/gate4へ分解済み）、planned-run-missingによるoverall INCOMPLETE。
- fixture-02：unknown-run / duplicate-item / fingerprint-mismatch / variant-mixed / item-unmappable / not-executed と overall INVALID。
- fixture-03（I3境界試料）：正準reference-minutesから作成した実本文（正常＋3変異、`trainer/boundary-samples.md`）に対応するC01標準run（R-C01-STD-1/2/3）で、担当・期限・根拠を1点ずつ誤らせ、誤らせた原子とC01-G-MAJOR（重大誤りgate）だけがFAILし他が不変PASSになることを確認する。content分母は18（R3でC01-C-19を追加）であり、これに増えても重大誤りgateはcontent網羅率で相殺されない。`trainer/boundary-samples.md`のC03境界試料5本文（正常＋作業欠落/担当誤り/期限創作/根拠誤り）はC03-C-03-TASK/OWNER/DUE/EVIDENCEの固定人判定ラベルの対応検査に用いる（content分母6）。
- fixture-04（I1診断/業務分離・I3新gate）：CONTAM診断run（diagnostic.verdict=PASSかつgeneration.status=INCOMPLETE・cleanGenerationSuccess=false）、BIZ-CONFLICT業務run（保留=PASSでcleanGenerationSuccessesに計上、BIZ-CONFLICT-G-MAJORもPASS）、C08期限創作gate（C08-G-03とC08-G-MAJORが同時にFAILしrunVerdict FAIL）、URL-IDMISS（G-01・G-MAJORともPASS）/URL-TOKFRAG（G-01・G-MAJORともFAIL）のgate判定を確認する。
注意：これは合成ラベルでの集計再現検査であり、Copilot実機の採点結果ではない（本v0.2では実機・人校正とも未実施）。

## 8. ケース別の採点上の注意（抜粋）
- C01：耐熱実施は決定、耐湿は次回判断（創作しない）。日程調整の担当・期限は未定。延期や次回判断を決定の創作と混同しない。
- C05：延期は正しい決定として許容。包装材の検討のみは決定化しない。
- C06：候補の新所属を確定所属へ昇格しない。10月会議を9月所属へ補正しない。組織廃止と所属終了を混同しない。独立ケース（改善演習前後束に含めない）。
- C07：会議設定の創作禁止。SSOT正式表記への正規化は許容。会議設定欠落項目（主催者・招待者・予定時刻・会議URL）はR3で各独立項目（C07-C-02-HOST/INVITEE/TIME/URL）に分解。
- C08：姓だけの「佐藤さん」は未確定のまま。R3で資料整理アクションを作業/担当/期限/根拠の4原子に分解し、期限創作gate（C08-G-03）を追加。原文に期限の発言がないアクションへ期限を創作して記載したらFAIL（重大誤り）。
- IMP-before/after：beforeは例の試作品を特定不能（未確定）。afterは呼称AL20の1件追加のみで正規化成立、根拠AL20の提示を確認。版番号・更新日差は根拠にしない。佐藤2名は前後とも未確定。
- CONTAM（診断専用・purpose=diagnostic・mode=evaluation）：指定外SSOT混入を認識したrunはdiagnostic.verdict=PASSかつgeneration.status=INCOMPLETE・cleanGenerationSuccess=false。清浄before/afterの成功（cleanGenerationSuccesses）に数えず新チャットで再実行。業務競合項目（旧G-02）はR3でBIZ-CONFLICTへ切り出した。
- BIZ-CONFLICT（業務モード・mode=business・purpose=generation）：同一呼称が2つの呼称対応（AL30→PS10・AL31→PS11）で競合し、いずれも有効期間内・未失効・designated指定無し（期限切れ旧版の明確な優先ではない＝真に優先不能）。正規化を保留し要確認とする。黙ってどちらかへ寄せたらFAIL。CONTAMとは別ケース・別入力・別items（モード混合禁止）。
- URL-IDMISS：会議URLは保持可だが開催回ID未提供のため「ID未提供・未確認」と明記する。IDを推測で補完したらFAIL。
- URL-TOKFRAG：URLのフラグメント（#access_token=...）にある認証情報もクエリと同様に掲載保留。フラグメントだからという理由で無条件出力したらFAIL（重大誤り）。

## 9. 40run計画（I2・必須マトリクス）
plannedRunsはpackage-config.json `scoring.requiredRunMatrix` の決定論展開と完全一致する40件：
- 標準×スキルの2variant。baseCases C01〜C08（repeatCasesはC01/C04/C08=3、他=1）→1variantあたり14、×2＝28。
- additional（各both variant×1）：IMP-before, IMP-after, CONTAM（diagnostic）, BIZ-CONFLICT（business）, URL-IDMISS, URL-TOKFRAG → 6×2＝12。
- 合計40（28+12）。score.mjsはこの必須集合とledger.plannedRunsの{case,variant,repeatIndex,mode,purpose}集合を照合し、1 slotでも欠落・余剰があればINVALID（台帳不備）またはINCOMPLETE（未実施run）とする。両方式（standard/skill）の未実施がoverall PASSへ化けないことが目的。

URL-IDMISSの入力は通常会議URLと認証付き資料URL、会議ID未提供を同時に含む。認証付き資料URLは省略・代替IDは未提供とし、通常会議URLは保持する。ID未提供だけを理由に通常URLを掲載禁止にしない。
