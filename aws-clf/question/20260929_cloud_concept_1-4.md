# 第2章 クラウドのコンセプト：2026-9-29の疑問1〜14

元のメモ：[2_cloud_concept.md](../cloud-practitioner-textbook/2_cloud_concept.md)の「2026-9-29」の疑問。回答日：2026-9-30。

## 1. 待機用データベースの稼働とコスト

**疑問**

可用性の確保のために、2台のデータベースサーバを常に同期させておくとあるが、コストが2倍になるような気がするが、1台は同期だけして、稼働はしていないのか？

**回答**

**待機側も同期のために稼働しており、費用がかかります。** 例えば、RDSの「メイン1台＋待機1台」のマルチAZ DBインスタンス構成では、待機側は普段アプリからの読み書きを担当せず、障害時にメインへ切り替わります。費用は増えますが、総額がちょうど2倍になるかは構成によります。停止時間を短くするための追加費用です。[RDS公式FAQ](https://aws.amazon.com/rds/faqs/)

## 2. 障害の予行演習を行う時期

**疑問**

障害発生時を想定した環境を構築とあるが、これはどの段階で行う？システム開発時か定期的に行うのか？

**回答**

**開発・公開前と、運用開始後の両方で行います。** 公開前に検証環境を用意し、監視や復旧手順が機能するか確認します。運用後も定期的に、また大きな構成変更の後に繰り返します。まず本番に近い検証環境で実施するのが基本です。[AWSの障害テストの指針](https://docs.aws.amazon.com/wellarchitected/latest/framework/rel_testing_resiliency_failure_injection_resiliency.html)、[定期的な演習の指針](https://docs.aws.amazon.com/wellarchitected/latest/framework/rel_testing_resiliency_game_days_resiliency.html)

## 3. 予行演習で再現する障害

**疑問**

実際に障害を発生させるとあるが、実際にどのような障害を発生させるのか？大きく分けてシステム起因とユーザ起因があるような気がする。

- システム起因：メモリリーク、ディスク容量不足など
- ユーザ起因：アクセス過多など

**回答**

例えば、**サーバーを1台停止する、通信を遮断・遅延させる、CPUやメモリを圧迫する**などです。監視が通知するか、予備サーバーへ切り替わるかを確認します。AWSには、こうした状況を再現する「AWS FIS」というサービスがあります。[AWS FIS公式資料](https://docs.aws.amazon.com/fis/latest/userguide/fis-actions-reference.html)

ご自身の分類に加えて、通信障害や管理者の設定ミスも対象になります。アクセス過多は正常な利用でも起こるので、「ユーザ起因」より**負荷増加**として整理すると分かりやすいです。

## 4. ユーザとSLAを結ぶ一般性

**疑問**

SLAを利用するユーザと結ぶのは一般的にあるのか？多いのか？少ないのか？

**回答**

**法人向けのクラウドやSaaSではよくあります。** AWSやGoogle WorkspaceにもSLAがあり、個別の契約書だけでなく、利用規約の一部として適用される形もあります。例えば「月間稼働率が基準を下回ったら、利用料金の一部を補填する」という約束です。AWS RDSでは、補填は原則として今後の利用料金に充てるクレジットです。[RDSのSLA](https://aws.amazon.com/rds/sla/)、[Google WorkspaceのSLA](https://workspace.google.com/terms/sla/)

## 5. アクセス急増の検知・予測とコスト

**疑問**

伸縮性とスケーラビリティについて、よく人気商品が発売される時に鯖落ちするが、これは技術的に検知することはできないのか？商品とインフラを結びつけるような技術は存在していない？予測システム的な。

- ご検知が起こった場合にちょっと怖い気がする。
  - 例えば、検知システムがアクセスの急増を予測して、サーバを増やしたは良いものの、結局アクセスが増えず、コストがかかるなど。
  - コストと見合わないからやっていないのか？
  - 最近だとAIも出てきて出来そうではあるが。。。
  - インフラ側が直接商品の人気度を知るのは密結合になりそうなので、何かしら監視システムと予測システムが必要そう。

**回答**

**検知・事前増強・予測の仕組みはあります。** 実際の負荷に応じて増やす[動的スケーリング](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scale-based-on-demand.html)、発売時刻に合わせて増やす[スケジュールスケーリング](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-scheduled-scaling.html)、機械学習で過去の負荷から予測する[予測スケーリング](https://aws.amazon.com/ec2/autoscaling/features/)があります。

ただし、突然の大ヒットは過去の傾向から予測しにくく、増設にも起動時間がかかります。サーバーを増やしても、DBなどが限界に達する可能性があります。

予測が外れた場合の費用対策は、**予測だけで精度を確認する・最大台数を決める・不要になった台数を減らす**ことです。予測スケーリングは増強を担当し、縮小には動的スケーリングなどを組み合わせます。[予測スケーリングの仕組み](https://docs.aws.amazon.com/autoscaling/ec2/userguide/predictive-scaling-policy-overview.html)

採用や増強量は、追加費用と停止による損失のバランスで決めます。設計例として、販売側から「発売日時・予想アクセス数」を増強の仕組みに渡せば、各サーバーが商品の詳細を直接知る必要はありません。

## 6. 同期処理と非同期処理のメリット・デメリット

**疑問**

同期処理と非同期処理のメリットデメリットは？

- 個人的な意見だが、非同期処理はバックエンド側としてはある程度メリットがありそうだが、フロントエンド側としては、デメリットもあるような気がする。
  - ユーザが自分の処理が完了しているのかが分かりづらいため。
  - ここはUX設計でうまくユーザを誘導させてあげることが大切であると感じる。
  - 例えば、メール配信。

**回答**

| 処理 | メリット | デメリット |
|---|---|---|
| 同期：結果を待つ | 結果やエラーをその場で確認しやすい | 相手が遅いと待たされ、障害の影響も受けやすい |
| 非同期：完了を待たずに進む | 早く受付を返せる。キューで負荷をならしやすい | 完了確認や再試行、重複処理の防止などの設計が必要 |

こうした特徴はAWSの[同期通信](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-integrating-microservices/synchronous.html)・[非同期通信](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-integrating-microservices/asynchronous.html)の資料でも説明されています。

ご自身の考えの通り、**非同期処理ではUX設計が大切です。** 例えばメール配信なら「受付済み→送信処理中→送信完了／失敗」を表示すると、状況が伝わります。

## 7. 並行処理のデメリット

**疑問**

並行処理のデメリットは？なんだか設計が複雑になりそう。。。

**回答**

**処理同士の調整が必要になり、設計が複雑になります。** 例えば、在庫1個の商品を2つの処理が同時に購入しようとしたら、二重販売を防ぐ仕組みが必要です。一部だけ失敗した場合の復旧や、結果をまとめる処理も必要になります。通信や調整の負担が増え、必ず速くなるとは限りません。[AWSの並列処理パターンと注意点](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/scatter-gather.html)

## 8. Well-Architected フレームワークの資料とPDF

**疑問**

Well-Architected フレームワークの資料のURLはどこにある？PDFとかでダウンロードできる？

**回答**

[日本語の公式資料](https://docs.aws.amazon.com/ja_jp/wellarchitected/latest/framework/welcome.html)で読めます。[日本語版PDF](https://docs.aws.amazon.com/ja_jp/wellarchitected/latest/framework/wellarchitected-framework.pdf)もダウンロードできます。

## 9. セキュリティの「完全性」

**疑問**

セキュリティの柱の「完全性」って何？

**回答**

**データを不正な変更や破損から守ることです。** 例えば、注文金額「1,000円」が第三者に「100円」へ書き換えられないようにすることです。[NISTの定義](https://csrc.nist.gov/glossary/term/integrity)

## 10. AWS Well-Architected Toolの使い方

**疑問**

AWS Well-Architected Toolは設計されたシステムに対して実行する診断ツールみたいなイメージで良いか？

**回答**

**チェックリストによる設計レビューのツール、というイメージが近いです。** 各柱の質問に、実施している対策を人が回答し、設計・運用のリスクや改善点を確認します。[AWS公式資料](https://docs.aws.amazon.com/wellarchitected/latest/userguide/continue-workflow-review.html)

## 11. セキュリティやデータ保護の規則の統制

**疑問**

「セキュリティやデータ保護などの規則が統制しやすい。」とはどのようなことか？

**回答**

**会社で決めたルールを、複数のAWS環境へまとめて適用・確認しやすいことです。** 例えば、AWS Organizationsで禁止する操作を設定し、AWS Configで設定のルール違反を検出できます。また、AWS CloudTrailで「誰が何を操作したか」を記録できます。[AWSの統制機能](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/organizations-security.html)、[CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)

## 12. AWSを利用したシステムの不正アクセス事例

**疑問**

AWSのサービスを利用したシステムにおいて、不正アクセスが確認された例はあるか？

- 最近、不正アクセスが増えてきているので、調査してみる。

**回答**

**あります。** 2019年の米Capital Oneの事件では、AWS上の環境の設定不備を悪用され、顧客情報が盗まれました。[米司法省の公表資料](https://www.justice.gov/usao-wdwa/pr/former-hacker-sentenced-stealing-computer-power-mine-cryptocurrency-and-stealing)

AWSを利用する場合も、利用者側で設定やアクセス権限を適切に管理する必要があります。これが**責任共有モデル**の考え方です。[AWS公式説明](https://aws.amazon.com/compliance/shared-responsibility-model/)

## 13. クラウド導入フレームワーク（CAF）の位置づけ

**疑問**

クラウド導入フレームワーク（CAF）は、クラウドを導入する組織のステークホルダーがそれぞれ知っているべき視点をまとめたものという認識で良いか？

**回答**

**はい、概ねその認識で合っています。** 関係者が何を準備・改善するかを6つの視点で整理し、組織全体のクラウド導入計画に役立てるガイドです。導入後の活用・改善にも使います。[AWS CAF公式資料](https://aws.amazon.com/cloud-adoption-framework/)

## 14. モダナイズ

**疑問**

モダナイズとは？

**回答**

**既存のシステムを、現在の技術や業務の要件に合うよう改善することです。** 例えば、自分で管理しているDBをRDSへ変更したり、大きなアプリを小さなサービスに分割したりします。運用負担を減らし、変更や拡張をしやすくすることが目的です。[AWSのモダナイズの説明](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-caf-platform-perspective/modern-apps.html)
