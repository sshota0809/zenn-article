---
title: "GCP の Cloud Load Balancing でマネージド証明書への切り替えを無停止で行う方法"
emoji: "🍀"
type: "tech" #
topics: ["GCP", "GCLB", "CloudLoadBalancing"]
published: true
---

# 概要

こんにちは、[@sshota0809](https://twitter.com/sshota0809) です。

最近、[Hybrid connectivity NEG を利用した、ハイブリッド/マルチクラウドでの負荷分散機能がサポートされたり](https://medium.com/google-cloud-jp/hybrid-load-balancing-27e77a4ec62) 、フロントのロードバランサをオンプレ環境から GCP の Cloud Load Balancing（GCLB）に移行するケースなどがより増えてくるのではないかと考えております。

せっかく、パブリッククラウドにロードバランサを移行するのであれば証明書管理からも解放されるべく、マネージド証明書も使いたいですよね。

しかし、既存のドメインを無停止でマネージド証明書をアタッチした GCLB に移行するには、実はひと工夫必要になります。

今回は、どのようにそれを実現するか紹介します。ちなみに、[本記事で紹介する内容は公式ドキュメントでもまとまっています](https://cloud.google.com/load-balancing/docs/ssl-certificates/google-managed-certs#migrating-ssl)。

# TL;DR

* GCP のマネージド証明書は GCLB にアタッチした上で[対象のドメインの DNS レコードを GCLB の IP に切り替えないと有効化されない](https://cloud.google.com/load-balancing/docs/ssl-certificates/troubleshooting#domain-status)
* DNS レコード設定〜マネージド証明書が有効化されるまで一定時間がかかるため、その時間証明書が利用できずその間ダウンタイムが発生してしまう
* 一方、[GCLB には複数の証明書をアタッチできる](https://cloud.google.com/load-balancing/docs/ssl-certificates#multiplessl) ため、同じ CN/SAN を持つセルフマネージド証明書とマネージド証明書をアタッチすることでダウンタイム無しで切り替えが可能
  * マネージド証明書が有効化されるまではセルフマネージド証明書が内部では利用される
  * マネージド証明書が有効化された後、セルフマネージド証明書をデタッチすることで以降マネージド証明書が利用される

# 課題
## GCP のマネージド証明書のアクティベート

GCP のマネージド証明書にはマネージドステータスとドメインステータスという概念があり、マネージド証明書の CN/SAN の DNS レコードがその証明書がアタッチされている IP アドレスで解決されないと `ACTIVE` ステータスにならず利用できません。

* 参考: [ドメインステータス](https://cloud.google.com/load-balancing/docs/ssl-certificates/troubleshooting#domain-status)
> **FAILED_NOT_VISIBLE**
> 
> ドメインの証明書のプロビジョニングが完了していません。次のいずれかが問題である可能性があります。
> ドメインの DNS レコードが、Google Cloud ロードバランサの IP アドレスに解決されません。この問題を解決するには、DNS の A レコードと AAAA レコードを更新して、ロードバランサの IP アドレスを指定します。
> DNS は、ロードバランサの IP アドレス以外の IP アドレスに解決されないようにする必要があります。たとえば、A レコードが正しいロードバランサに解決され、AAAA は別のロードバランサに解決される場合、ドメインのステータスは FAILED_NOT_VISIBLE になります。
> 
> 更新された DNS A レコードと AAAA レコードが完全に伝播されるまで、かなり時間がかかることがあります。通常は数時間程度ですが、インターネットを介した反映には最長で 72 時間かかることもあります。ドメイン ステータスは、伝播が完了するまで FAILED_NOT_VISIBLE になります。
> 
> SSL 証明書がロードバランサのターゲット プロキシに適用されていません。この問題を解決するには、ロードバランサの構成を更新してください。
> 
> グローバル転送ルール用のフロントエンド ポートに、SSL プロキシ ロードバランサ用のポート 443 が含まれていません。この問題は、ポート 443 を使用して新しい転送ルールを追加すると解決できます。
> 
> マネージド ステータスが PROVISIONING の場合、ドメイン ステータスが FAILED_NOT_VISIBLE であっても、Google Cloud はプロビジョニングを再試行します。

新しくシステムとしてサービスインするドメインであれば DNS レコードを登録してアクティベートされるのを待てばよいですが、既にサービスイン済みの DNS レコードの切り替えを行ってしまうとマネージド証明書がアクティベートされるまでサービスは停止してしまいます。だからといって DNS レコードを切り替えないと永遠とマネージド証明書はアクティベートされないので、このままではサービスのダウンタイムは避けられません。

# 解決
## GCLB に複数の証明書をアタッチする

一方、GCLB に複数の証明書をアタッチすることができます。これは、同一 IP アドレス/ポートで複数のドメインに対してサービス提供をするためです。

* 参考: [複数の SSL 証明書](https://cloud.google.com/load-balancing/docs/ssl-certificates#multiplessl)
> ターゲット HTTPS またはターゲット SSL プロキシあたりの SSL 証明書の最大数を構成できます。同じロードバランサの IP アドレスとポートを使用して複数のドメインからサービスを提供し、ドメインごとに異なる SSL 証明書を使用する場合は、複数の SSL 証明書を使用します。

この仕様は、上記公式ドキュメントの通り複数のドメインに対してサービス提供をするためですが、実は同一 CN/SAN の証明書を複数アタッチしても GCLB 上でエラーが出たりはしません。同一 CN/SAN の証明書がアタッチされた場合の仕様は下記の通り上記ドキュメントでも解説されており、SNI に一致する証明書のうち 1 つが利用されます。

> SNI ホスト名が複数の証明書の CN または SAN と一致する場合、証明書の選択はクライアント固有の内部要因に基づいて行われますが、これは予測不能です。SNI に一致する証明書の 1 つが返されます。期限切れの証明書がまだターゲット プロキシに関連付けられている場合、ロードバランサが期限切れの証明書を提供することもあります。

この、同一 CN/SAN の証明書を複数アタッチできる仕様を上手く利用することで、ダウンタイムを回避しつつ GCLB にサービスを切り替えすることができます。

## セルフマネージド証明書とマネージド証明書をアタッチする

ダウンタイムを回避しつつサービスを切り替える方法を簡単に手順化すると下記の通りになります。

1. 利用中の既存の証明書もしくは Let's Encrypt で発行したセルフマネージド証明書を GCP に登録する
2. 1 と同じ CN/SAN のマネージド証明書を GCP で作成する
3. 1 と 2 両方の証明書を GCLB にアタッチする
4. 対象ドメインの DNS レコードを GCLB の IP に切り替える
5. 2 で作成したマネージド証明書が `ACTIVE` になったら 1 のセルフマネージド証明書をデタッチする

仕組みとしては、手順 4 の段階ではマネージド証明書はアクティベートされていませんが、セルフマネージド証明書がアタッチされているため GCLB ではセルフマネージド証明書を利用して通信されます。これによって DNS レコード切り替え〜マネージド証明書アクティベートの間もダウンタイム発生を防ぐことができます。後は、マネージド証明書が `ACTIVE` になってアクティベートされたらセルフマネージド証明書をデタッチすることで、それ以降はマネージド証明書が利用されるようになります。（正確にはマネージド証明書がアクティベートされたタイミングの時点でそちらが利用されるかもしれません。）

ちなみに、Terraform の公式ドキュメントを見ても SSL 証明書は `List` 形式で指定が可能となっていますし、GKE の Ingress 経由で設定を行う場合も`カンマ`区切りで証明書を指定することができます。

* 参考: [google_compute_target_https_proxy](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_target_https_proxy#ssl_certificates)
> ssl_certificates - (Required) A list of SslCertificate resources that are used to authenticate connections between users and the load balancer. At least one SSL certificate must be specified.

* 参考: [複数の SSL 証明書](https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-multi-ssl#pre-shared-certs)
> ingress.gcp.kubernetes.io/pre-shared-cert: "FIRST_CERT_NAME,SECOND_CERT_NAME"

# 終わりに
ハイブリッド/マルチクラウドでの負荷分散機能もサポートされましたしダウンタイムも発生しませんし、これでどんどん GCLB にロードバランシングの機能を移行していくことができますね。 （現在所属している組織では、実際に現在オンプレで稼働しているグローバルの入り口とロードバランシング機能を GCLB に移行し始めたりしています。）
