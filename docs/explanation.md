# Codex OTel Local Demo 解説

このドキュメントは、今回作ったローカルデモを「何も知らない状態から読める」ように説明したものです。コマンド手順だけ知りたい場合は `README.md`、仕組みを理解したい場合はこちらを読む、という位置づけです。

## まず何を作ったのか

Codexが出すテレメトリデータを、自分のPCの中だけで受け取り、Grafanaで見られるようにしました。

ざっくり言うと、こういう流れです。

```text
Codex
  -> OpenTelemetry Collector
    -> Loki       -> Grafanaでログを見る
    -> Prometheus -> Grafanaでメトリクスを見る
```

Codexは「何が起きたか」をOpenTelemetry形式で送ります。OpenTelemetry Collectorがそれを受け取り、ログはLokiへ、メトリクスはPrometheusへ渡します。GrafanaはLokiとPrometheusを読みに行って、人間が見やすい画面にします。

## OpenTelemetryとは

OpenTelemetry、よくOTelと略されるものは、アプリケーションの状態や動きを外に出すための共通規格です。

たとえばアプリを動かしていると、知りたいことがいろいろあります。

- どんな処理が実行されたか
- どれくらい時間がかかったか
- どのツールやAPIが呼ばれたか
- エラーが起きたか
- リクエスト数や処理時間が増えているか

こういう観測用データを、毎回アプリごとに独自形式で出すと扱いが大変です。OTelは「ログ、メトリクス、トレースをこういう形式で出そう」という共通の約束を提供します。

今回のCodexでも、OTel出力を有効にするとCodex内で起きたイベントやメトリクスが外へ送られます。

## 3種類のテレメトリ

OTelでよく出てくるデータは、大きく3種類あります。

**Logs**

「この時刻にこういうイベントが起きた」という記録です。今回一番わかりやすいのはこれです。

例:

```text
codex.user_prompt
codex.tool_decision
codex.tool_result
codex.api_request
```

ただし今回の設定では `log_user_prompt = false` にしています。つまり、ユーザーが入力したプロンプト本文そのものは送らず、長さなどのメタデータ中心になります。これは安全寄りの設定です。

**Metrics**

数値として集計できるデータです。回数、時間、サイズなどです。

例:

```text
APIリクエスト数
APIリクエスト時間
tool callの回数
turnごとの処理時間
```

Prometheusではメトリクス名にドットが使えないため、OTel側の `codex.api_request` のような名前は `codex_api_request` のようにアンダースコアへ変換されます。counterの場合は `_total` が付くこともあります。

**Traces**

1つの処理の流れを分解して見るためのデータです。たとえば「ユーザー入力を受ける -> モデルへ問い合わせる -> ツールを呼ぶ -> 結果を返す」のような流れを、spanという単位で追えます。

今回のデモでは、まずLogsとMetricsに絞っています。Tracesは次の拡張候補です。

## 登場人物

今回のComposeには4つのサービスがあります。

**OpenTelemetry Collector**

ファイル: `otel-collector-config.yaml`

CodexからOTelデータを受け取る入口です。今回のCollectorは2つの口を開けています。

```text
4317: OTLP/gRPC
4318: OTLP/HTTP
```

Codexからは `http://localhost:4318/v1/logs` や `http://localhost:4318/v1/metrics` に送ります。

Collectorは受け取ったデータを整えて、後ろの保存先へ流します。

**Loki**

ログ保存用のサーバーです。Grafana Labs製で、Grafanaから読みやすいログストアです。

今回のログはCollectorからLokiへ送られます。GrafanaのExploreでLoki datasourceを選ぶと、Codexのイベントログを検索できます。

**Prometheus**

メトリクス保存用のサーバーです。定期的にHTTP endpointを見に行き、そこに出ている数値を集めます。

今回、Collectorは受け取ったメトリクスを `http://localhost:9464/metrics` にPrometheus形式で公開します。Prometheusはそれを5秒ごとに取りに行きます。

**Grafana**

可視化画面です。Grafana自体がデータを保存しているわけではなく、LokiやPrometheusに問い合わせて表示します。

今回のGrafanaでは、datasourceを自動登録しています。

```text
Loki       -> logs用
Prometheus -> metrics用
```

## 今回のデータの流れ

Codexをこのコマンドで起動します。

```sh
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 \
OTEL_EXPORTER_OTLP_LOGS_ENDPOINT=http://localhost:4318/v1/logs \
OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://localhost:4318/v1/metrics \
OTEL_METRIC_EXPORT_INTERVAL=5000 \
codex \
  -c 'otel.environment="local"' \
  -c 'otel.exporter={ otlp-http = { endpoint = "http://localhost:4318/v1/logs", protocol = "binary" } }' \
  -c 'otel.log_user_prompt=false' \
  -c 'otel.metrics_exporter={ otlp-http = { endpoint = "http://localhost:4318/v1/metrics", protocol = "binary" } }'
```

重要なのはこの2つです。

```text
logs    -> http://localhost:4318/v1/logs
metrics -> http://localhost:4318/v1/metrics
```

どちらもCollectorのOTLP/HTTP receiverに届きます。

Collector側では、この設定で受けています。

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
```

そして、ログとメトリクスを別々のpipelineに流しています。

```yaml
service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [resource/codex_demo, batch]
      exporters: [otlp_http/loki, debug]
    metrics:
      receivers: [otlp]
      processors: [resource/codex_demo, batch]
      exporters: [prometheus, debug]
```

`debug` exporterも入れているので、Collectorのログを見るだけでも「受け取れているか」を確認できます。

```sh
docker compose logs -f otel-collector
```

## なぜservice_nameがcodex-local-demoになるのか

Collectorでこのprocessorを入れています。

```yaml
processors:
  resource/codex_demo:
    attributes:
      - key: service.name
        value: codex-local-demo
        action: upsert
      - key: deployment.environment
        value: local
        action: upsert
```

これは受け取ったテレメトリにラベルを付ける処理です。

GrafanaやLoki/Prometheusで検索するとき、ラベルがないと探しにくくなります。そこで全部に `service.name=codex-local-demo` と `deployment.environment=local` を付けています。

LokiやPrometheusでは、ドットがアンダースコアに変わって見えることがあります。

```text
service.name -> service_name
deployment.environment -> deployment_environment
```

そのためGrafanaではこう検索します。

```logql
{service_name="codex-local-demo"}
```

## Grafanaで何を見るか

まずはGrafanaを開きます。

```text
http://localhost:3000
```

最初はDashboardよりExploreがおすすめです。Exploreは「いま何が入っているか」を直接探す画面です。

**ログを見る**

左メニューからExploreを開き、datasourceで `Loki` を選びます。

まずこのクエリを入れます。

```logql
{service_name="codex-local-demo"}
```

Codexのイベントだけに寄せたい場合はこうします。

```logql
{service_name="codex-local-demo"} |= "codex."
```

スモークテストを流した場合はこれでも見られます。

```logql
{event_name="codex.otel_smoke_test"}
```

**メトリクスを見る**

Exploreでdatasourceを `Prometheus` に切り替えます。

まずはこれです。

```promql
{service_name="codex-local-demo"}
```

スモークテストmetricならこれです。

```promql
codex_otel_smoke_metric_total
```

Codex実行後は、Codexのバージョンや実行内容によって名前が変わる可能性があります。候補としてはこういう名前を探します。

```text
codex_api_request
codex_api_request_duration_ms
codex_tool_call
codex_tool_call_duration_ms
codex_turn_*
```

## DashboardとExploreの違い

Grafanaには大きく2つの見方があります。

**Explore**

手でクエリを書いて、その場で探す画面です。デバッグや学習に向いています。

今回なら「本当にログが届いた？」「metric名は何になった？」を見るのに便利です。

**Dashboard**

よく見るグラフやログパネルを固定しておく画面です。運用や継続観察に向いています。

今回のデモでは `Codex OTel Overview` という簡単なdashboardを用意しています。ただし、最初はExploreの方がわかりやすいです。metric名がCodexのバージョンで変わっても、Exploreならその場で探せます。

## スモークテストとは

スモークテストは「本物のCodexを動かす前に、配管だけ通っているか確認する小さなテスト」です。

今回READMEにあるスモークテストでは、手動でCollectorへ小さなログとメトリクスを送っています。

ログの流れ:

```text
curl
  -> Collector /v1/logs
    -> Loki
      -> Grafana Explore
```

メトリクスの流れ:

```text
curl
  -> Collector /v1/metrics
    -> Collector /metrics
      -> Prometheus
        -> Grafana Explore
```

これが成功していれば、少なくともCollector、Loki、Prometheus、Grafanaの配管は正しく動いています。

そのうえでCodexのテレメトリが見えない場合は、問題は主にCodex側の設定や、Codexがまだ該当イベントをflushしていないことに絞れます。

## よくある詰まりどころ

**何も表示されない**

まず時間範囲を確認します。Grafana右上の時間範囲が短すぎると、さっき送ったデータが範囲外になることがあります。`Last 30 minutes` くらいにします。

**Codexを動かしたのにログが出ない**

Codexはテレメトリをbatchして非同期に送ります。セッションを終了したタイミングでflushされることがあります。まずCodexを終了してからGrafanaを見ます。

**Prometheusでmetricが見えない**

PrometheusはCollectorの `/metrics` を定期的に取りに行きます。送信直後はまだscrapeされていないことがあります。数秒待ってから再検索します。

また、名前が変換されることがあります。

```text
codex.otel_smoke_metric -> codex_otel_smoke_metric_total
```

**Lokiではservice.nameではなくservice_nameを使う**

OTelの属性名 `service.name` は、LokiやPrometheusでは `service_name` のように見えることがあります。今回のGrafanaクエリでは `service_name` を使います。

**Dockerは起動しているのにCodexからlocalhostへ送れない**

今回のCodexはホスト側で動き、CollectorはDockerのport mappingでホストの `localhost:4318` に公開されています。したがってCodex設定では `http://localhost:4318/...` を使います。

Dockerコンテナ同士の通信では `otel-collector:4318` のようなサービス名を使えますが、ホストで動くCodexからは `localhost` が正解です。

## もう少し深い理解メモ

ここからは、このデモを触りながら理解したOTelまわりの補足です。最初は読み飛ばしても大丈夫ですが、Grafanaで実際のデータを見始めると効いてきます。

**OTLPの `/v1/logs` と `/v1/metrics` は標準パス**

Codex起動時には、logsとmetricsの送信先をこう指定しています。

```sh
-c 'otel.exporter={ otlp-http = { endpoint = "http://localhost:4318/v1/logs", protocol = "binary" } }'
-c 'otel.metrics_exporter={ otlp-http = { endpoint = "http://localhost:4318/v1/metrics", protocol = "binary" } }'
```

`/v1/logs` や `/v1/metrics` はこのリポジトリ独自のパスではなく、OTLP/HTTPの標準パスです。Collector側では `4318` でOTLP/HTTP receiverを立てているだけです。

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
```

OTLP/HTTPでは、signalごとに標準パスが分かれます。

```text
POST /v1/logs
POST /v1/metrics
POST /v1/traces
```

**Collectorの `exporter` は出口**

Collector設定には、`receivers`, `processors`, `exporters`, `service` が出てきます。

```text
receiver  = 入り口
processor = 加工
exporter  = 出口
service   = 部品をどう配線して動かすか
```

今回のexporter定義はこうです。

```yaml
exporters:
  debug:
    verbosity: normal
  otlp_http/loki:
    endpoint: http://loki:3100/otlp
  prometheus:
    endpoint: 0.0.0.0:9464
```

`debug`, `otlp_http`, `prometheus` はCollectorが知っているexporter typeです。`otlp_http/loki` の `/loki` は任意のインスタンス名で、「Loki向けに使っているotlp_http exporter」と区別するための名前です。

ただし、exporterを定義しただけでは使われません。どのsignalをどのexporterへ流すかは `service.pipelines` で決まります。

```yaml
service:
  pipelines:
    logs:
      exporters: [otlp_http/loki, debug]
    metrics:
      exporters: [prometheus, debug]
```

つまり「ログはLokiへ、メトリクスはPrometheusへ」は `service.pipelines` が持っている配線です。

**Codexのevent名はログ本文ではなくstructured metadataに出る**

Codexのドキュメントには、`codex.tool_decision` や `codex.tool_result` のようなイベント名が出てきます。これは「Lokiのログ本文にその文字列が表示される」という意味ではなく、OTel LogRecordの属性としてイベント名が入る、という意味です。

Loki/Grafanaでは、ログ本文が空に見えることがあります。その場合でも、行を展開するとStructured metadataに次のような値が入っています。

```text
event_name = codex.tool_decision
decision = approved
event_timestamp = ...
duration_ms = ...
success = true
```

Grafana Exploreでは、本文検索よりもmetadata filterを使うと探しやすいです。

```logql
{service_name="codex-local-demo"} | event_name="codex.tool_decision"
```

tool系をまとめて見るなら:

```logql
{service_name="codex-local-demo"} | event_name=~"codex\\.tool_.*"
```

ログ一覧で見やすくしたい場合は `line_format` を使います。

```logql
{service_name="codex-local-demo"}
| event_name=~"codex\\.tool_(decision|result)"
| line_format "{{.event_timestamp}} {{.event_name}} tool={{.tool_name}} decision={{.decision}} success={{.success}} duration={{.duration_ms}}ms"
```

**labelsとstructured metadataは違う**

Lokiにはlabelsとstructured metadataがあります。

```text
labels
  インデックスされる検索キー。少数に絞る。
  例: service_name, deployment_environment

structured metadata
  各ログ行に付く構造化フィールド。
  例: event_name, model, duration_ms, success, decision
```

今回のLokiでは、`service_name` と `deployment_environment` がlabelsとして見え、`event_name` や `decision` などはstructured metadataとして見えます。

**LogRecord、Semantic Conventions、Codex独自スキーマは別レイヤー**

OTelのLogRecordは「1件のログ/イベントの標準的な入れ物」です。大まかには次のような枠を持ちます。

```text
Timestamp
SeverityText / SeverityNumber
Body
Resource
Attributes
TraceId / SpanId
```

Semantic Conventionsは「共通の意味を持つ属性は、この名前で入れよう」という標準辞書です。

```text
service.name
service.version
host.name
telemetry.sdk.language
exception.type
log.file.name
```

一方、Codex固有のイベント名や属性はCodex独自スキーマです。

```text
event_name = codex.tool_decision
decision = approved
tool_name = ...
conversation_id = ...
```

つまり:

```text
OTelデータモデル
  ログやメトリクスの箱の形

Semantic Conventions
  共通概念の標準的な属性名

Codex独自スキーマ
  Codexがemitする固有イベントや固有属性
```

**属性を入れるかどうかは、作る側や収集する側が決める**

たとえばLog Semantic Conventionsに `log.file.name` があります。これは「ログがファイル由来なら、そのファイル名をこの属性名で入れよう」という約束です。

ただし、OTelが勝手に `log.file.name` を入れるわけではありません。実際に属性を入れるのは、そのLogRecordを作る側です。

```text
アプリが直接OTelをemitする場合
  アプリやSDKが属性を決める

Collectorのfilelog receiverがファイルを読む場合
  receiver/operatorが log.file.name などを付ける

HTTP自動計装の場合
  instrumentation libraryが http.request.method などを付ける

Collector processorで補う場合
  設定で service.name などを追加する
```

今回こちらで明示的に足しているのは、Collector processorのこの部分です。

```yaml
processors:
  resource/codex_demo:
    attributes:
      - key: service.name
        value: codex-local-demo
        action: upsert
      - key: deployment.environment
        value: local
        action: upsert
```

**Skill注入はPrometheus metricで見る**

Codexのskillは、必要だと判断されるとモデルに渡すコンテキストへ入ります。これを「skillが注入される」と考えるとわかりやすいです。

```text
Skill injected
  = そのskillの説明や指示がモデル入力に入った
  = ただし、モデルが実際に従った証明ではない
```

Codexはskill注入をログではなくメトリクスとしてemitします。Prometheusでは次のmetricで見えます。

```promql
codex_skill_injected_total
```

特定skillを見るなら:

```promql
codex_skill_injected_total{skill="baseline-ui"}
```

直近30分でskill別に集計するなら:

```promql
sum by (skill, status, invoke_type) (
  increase(codex_skill_injected_total[30m])
)
```

このmetricでわかるのは「そのskillがモデル入力に入った」ことです。一方で、「モデルがそのskillの指示に実際に従ったか」までは直接証明できません。そこを見るには、tool call、出力形式、レビュー結果など別の観測可能な契約が必要になります。

ログとメトリクスの境界は、少し直感に反することがあります。たとえば「skillが注入された」は出来事なのでログっぽく感じますが、Codexでは「どのskillが何回注入されたか」を集計しやすいcounter metricとして扱っています。

```text
codex.tool_decision
  1件ごとの詳細を追いたい
  -> LogRecord
  -> Loki

skill.injected
  回数や傾向を集計したい
  -> Metric counter
  -> Prometheus
```

## ファイルごとの役割

`docker-compose.yml`

4つのサービスを起動します。

- `otel-collector`
- `loki`
- `prometheus`
- `grafana`

`otel-collector-config.yaml`

OTel Collectorの設定です。今回の一番大事なファイルです。入口、加工、出口を決めています。

`prometheus/prometheus.yml`

Prometheusがどこをscrapeするかを書いています。今回はCollectorの `otel-collector:9464` を見に行きます。

`loki/local-config.yaml`

Lokiをローカル単体で動かすための設定です。

`grafana/provisioning/datasources/datasources.yml`

GrafanaにLokiとPrometheusを自動登録します。

`grafana/provisioning/dashboards/dashboards.yml`

Grafanaにdashboardファイルを読み込ませます。

`grafana/dashboards/codex-otel-overview.json`

簡単なCodex OTel dashboardです。

`codex-config.sample.toml`

Codex側に永続設定したい場合のサンプルです。ただし普段の設定を汚さないため、最初はREADMEの一時起動コマンドを使うのがおすすめです。

## 今回理解できれば十分なこと

最初は細かい設定を全部覚えなくて大丈夫です。まずはこの4点だけ押さえれば十分です。

1. CodexはOTel形式でログやメトリクスを出せる。
2. CollectorはOTelデータの受付係で、保存先へ振り分ける。
3. Lokiはログ、Prometheusはメトリクスを保存する。
4. GrafanaはLokiとPrometheusを見に行って表示する。

つまり、今回作ったものは「Codexの中で起きたことを、ローカルで観察するための小さな監視基盤」です。
