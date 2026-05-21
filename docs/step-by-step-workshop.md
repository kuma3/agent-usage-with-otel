# Codex OTel Local Demo Step-by-Step Workshop

このドキュメントは、CodexのOpenTelemetry出力をローカルで受け取り、Grafanaで可視化するまでを、手を動かしながら理解するための実装教材です。

完成形だけを知りたい場合は `README.md`、仕組みをざっくり読みたい場合は `docs/explanation.md` を見てください。このドキュメントは「なぜそのファイルを作るのか」「作ったあと何を確認するのか」を順番に追うためのものです。

## 0. 今回やりたいこと

最近のCoding Agentは、OpenTelemetry、略してOTel形式でテレメトリデータを出せるようになっています。

テレメトリとは、アプリケーションやツールが動いているときに外へ出す観測用データです。たとえば次のようなものです。

- どんなイベントが起きたか
- APIリクエストが何回発生したか
- ツール呼び出しにどれくらい時間がかかったか
- エラーが起きたか

今回のゴールは、Codexから出たOTelデータをローカルPC内で受け取り、Grafanaで見ることです。

完成時の流れはこうです。

```text
Codex
  -> OpenTelemetry Collector
    -> Loki       -> Grafanaでログを見る
    -> Prometheus -> Grafanaでメトリクスを見る
```

ポイントは、クラウドサービスを使わずに全部ローカルで試すことです。

## 1. 必要なもの

この教材では次のものを使います。

- Docker Desktop
- Docker Compose
- Codex CLI
- ブラウザ

確認コマンド:

```sh
docker --version
docker compose version
codex --version
```

Docker Desktopが起動しているかも確認します。

```sh
docker ps
```

ここでコンテナ一覧が表示されればOKです。Docker daemonに接続できない場合は、Docker Desktopを起動してください。

## 2. プロジェクトを作る

作業ディレクトリを作ります。

```sh
mkdir agent-usage-with-otel
cd agent-usage-with-otel
```

今回作るファイルは、おおまかに次のようになります。

```text
.
├── docker-compose.yml
├── otel-collector-config.yaml
├── prometheus/
│   └── prometheus.yml
├── loki/
│   └── local-config.yaml
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── datasources.yml
│   │   └── dashboards/
│   │       └── dashboards.yml
│   └── dashboards/
│       └── codex-otel-overview.json
├── codex-config.sample.toml
└── README.md
```

最初から全部理解しなくて大丈夫です。順番に作っていきます。

## 3. Docker Composeで全体構成を定義する

まず、4つのサービスをDocker Composeで起動できるようにします。

作るサービスは次の4つです。

**otel-collector**

CodexからOTelデータを受け取る入口です。

**loki**

ログを保存します。

**prometheus**

メトリクスを保存します。

**grafana**

LokiとPrometheusを読みに行き、画面で表示します。

`docker-compose.yml` を作ります。

```yaml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.147.0
    command: ["--config=/etc/otelcol/config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol/config.yaml:ro
    ports:
      - "4317:4317"
      - "4318:4318"
      - "9464:9464"
    depends_on:
      - loki

  prometheus:
    image: prom/prometheus:v3.7.3
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--web.enable-lifecycle"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    depends_on:
      - otel-collector

  loki:
    image: grafana/loki:3.6.2
    command: ["-config.file=/etc/loki/local-config.yaml"]
    volumes:
      - ./loki/local-config.yaml:/etc/loki/local-config.yaml:ro
      - loki-data:/loki
    ports:
      - "3100:3100"

  grafana:
    image: grafana/grafana:12.3.0
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
      GF_AUTH_DISABLE_LOGIN_FORM: "true"
      GF_USERS_DEFAULT_THEME: light
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
      - loki

volumes:
  grafana-data:
  loki-data:
  prometheus-data:
```

ここで重要なのはportです。

```text
4317: OTel Collector OTLP/gRPC
4318: OTel Collector OTLP/HTTP
9464: CollectorがPrometheus向けに公開するmetrics endpoint
9090: Prometheus
3100: Loki
3000: Grafana
```

Codexからはホスト側の `localhost:4318` に送ります。

## 4. OpenTelemetry Collectorを設定する

次にCollectorの設定を作ります。

Collectorは、OTelデータの受付係兼ルーターです。

やることは3つです。

1. CodexからOTLPを受け取る
2. 共通ラベルを付ける
3. logsはLokiへ、metricsはPrometheus向けendpointへ流す

`otel-collector-config.yaml` を作ります。

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  resource/codex_demo:
    attributes:
      - key: service.name
        value: codex-local-demo
        action: upsert
      - key: deployment.environment
        value: local
        action: upsert
  batch:

exporters:
  debug:
    verbosity: normal
  otlp_http/loki:
    endpoint: http://loki:3100/otlp
  prometheus:
    endpoint: 0.0.0.0:9464
    resource_to_telemetry_conversion:
      enabled: true

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

設定の読み方です。

`receivers` は入口です。

```yaml
receivers:
  otlp:
```

ここで `4317` と `4318` を開きます。

`processors` は加工です。

```yaml
service.name = codex-local-demo
deployment.environment = local
```

というラベルを付けます。Grafanaで探しやすくするためです。

`exporters` は出口です。

```text
debug          -> Collectorのログに出す
otlp_http/loki -> Lokiへ送る
prometheus     -> /metrics を公開する
```

`service.pipelines` は、どの入口から入ったデータを、どの加工を通して、どの出口に出すかを決めます。

## 5. Prometheusを設定する

Prometheusは、自分から定期的にHTTP endpointを取りに行くタイプのメトリクス保存サーバーです。この取りに行く動きをscrapeと呼びます。

Collectorは `9464` でPrometheus形式のメトリクスを公開します。Prometheusにはそこを見に行かせます。

`prometheus/prometheus.yml` を作ります。

```yaml
global:
  scrape_interval: 5s
  evaluation_interval: 5s

scrape_configs:
  - job_name: otel-collector
    static_configs:
      - targets:
          - otel-collector:9464
```

Docker Compose内では、サービス名 `otel-collector` でCollectorにアクセスできます。そのためPrometheusから見る宛先は `otel-collector:9464` です。

一方、ホストで動くCodexから見る宛先は `localhost:4318` です。

この違いは少し大事です。

```text
ホストで動くCodex      -> localhost:4318
Compose内のPrometheus -> otel-collector:9464
```

## 6. Lokiを設定する

Lokiはログ保存用のサーバーです。

`loki/local-config.yaml` を作ります。

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  filesystem:
    directory: /loki/chunks

limits_config:
  allow_structured_metadata: true
  volume_enabled: true
  retention_period: 24h

compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  delete_request_store: filesystem

ruler:
  storage:
    type: local
    local:
      directory: /loki/rules
```

今回の目的はローカルデモなので、認証なし、単体構成、ローカルファイル保存で十分です。

## 7. Grafanaのdatasourceを自動登録する

GrafanaはLokiとPrometheusのデータを表示する画面です。

Grafana上で手動設定してもよいのですが、今回は起動したらすぐ使えるようにprovisioningを使います。

`grafana/provisioning/datasources/datasources.yml` を作ります。

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    uid: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true

  - name: Loki
    uid: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: true
```

ここでもCompose内通信なので、GrafanaからPrometheusへは `http://prometheus:9090`、Lokiへは `http://loki:3100` でアクセスします。

## 8. Grafana dashboardを登録する

Exploreだけでも十分ですが、最初から簡単なdashboardも置いておきます。

まずdashboard providerを作ります。

`grafana/provisioning/dashboards/dashboards.yml`:

```yaml
apiVersion: 1

providers:
  - name: Codex OTel
    orgId: 1
    folder: Codex OTel
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
```

実際のdashboard JSONは `grafana/dashboards/codex-otel-overview.json` に置きます。

教材としては、dashboard JSONを手書きで理解する必要はありません。Grafana dashboardはJSONとしてexport/importできる、という理解で十分です。

最初に見るならdashboardよりExploreがおすすめです。Exploreの方が「どんなログやmetricが入っているか」を探しやすいからです。

## 9. Codex側の設定サンプルを作る

次にCodex側の設定例を置きます。

重要な方針として、今回のデモでは `~/.codex/config.toml` を直接書き換えません。普段のCodex利用に影響しないようにするためです。

`codex-config.sample.toml`:

```toml
[otel]
environment = "local"
exporter = { otlp-http = { endpoint = "http://localhost:4318/v1/logs", protocol = "binary" } }
log_user_prompt = false
metrics_exporter = { otlp-http = { endpoint = "http://localhost:4318/v1/metrics", protocol = "binary" } }
```

`log_user_prompt = false` は大事です。これにより、ユーザーが入力したプロンプト本文そのものは送られません。ローカルデモでも、まずは安全寄りの設定にします。

実際にCodexを起動するときは、ファイルに書く代わりに `-c` オプションで一時的に渡します。

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

ここで一度つまずいたポイントがあります。

`otel.metrics_exporter` は文字列ではなく、inline table形式で渡す必要がありました。

動かない例:

```sh
-c 'otel.metrics_exporter="otlp-http"'
```

動く例:

```sh
-c 'otel.metrics_exporter={ otlp-http = { endpoint = "http://localhost:4318/v1/metrics", protocol = "binary" } }'
```

エラーとしては、次のようなものが出ます。

```text
Error loading config.toml: invalid type: unit variant, expected struct variant
in `otel.metrics_exporter`
```

このエラーは「単なる名前ではなく、endpointなどを含む構造を期待している」という意味です。

## 10. Compose設定を検証する

ファイルを作ったら、まずCompose設定が正しいか確認します。

```sh
docker compose config
```

静かに成功だけ確認したい場合:

```sh
docker compose config --quiet
```

ここでエラーが出たら、YAMLのインデントやファイルパスを確認します。

## 11. スタックを起動する

起動します。

```sh
docker compose up -d
```

状態を確認します。

```sh
docker compose ps
```

期待する状態は、次の4つが `Up` になっていることです。

```text
grafana
loki
otel-collector
prometheus
```

それぞれの画面やendpointは次の通りです。

```text
Grafana    http://localhost:3000
Prometheus http://localhost:9090
Loki       http://localhost:3100
Collector  http://localhost:4318
```

## 12. ヘルスチェックする

Grafana:

```sh
curl http://localhost:3000/api/health
```

Prometheus:

```sh
curl http://localhost:9090/-/ready
```

Loki:

```sh
curl http://localhost:3100/ready
```

PrometheusがCollectorをscrapeできているか:

```sh
curl http://localhost:9090/api/v1/targets
```

`otel-collector:9464` のhealthが `up` ならOKです。

## 13. Codexなしでログの配管をテストする

いきなりCodexを動かす前に、まず手動で小さなOTLPログを送ります。

```sh
curl -X POST http://localhost:4318/v1/logs \
  -H 'Content-Type: application/json' \
  -d '{"resourceLogs":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"manual-otel-smoke"}}]},"scopeLogs":[{"logRecords":[{"severityText":"INFO","body":{"stringValue":"codex otel smoke test"},"attributes":[{"key":"event.name","value":{"stringValue":"codex.otel_smoke_test"}}]}]}]}]}'
```

成功すると、次のようなレスポンスになります。

```json
{"partialSuccess":{}}
```

Collectorのdebug exporterにも出るはずです。

```sh
docker compose logs --tail=80 otel-collector
```

Lokiにも入っているか確認します。

```sh
curl --get http://localhost:3100/loki/api/v1/query_range \
  --data-urlencode 'query={service_name="codex-local-demo"} |= "codex otel smoke test"' \
  --data-urlencode 'limit=5'
```

ここで結果が返れば、ログの流れは通っています。

```text
curl -> Collector -> Loki -> Grafana
```

## 14. Codexなしでメトリクスの配管をテストする

次に、小さなOTLPメトリクスを送ります。

```sh
curl -X POST http://localhost:4318/v1/metrics \
  -H 'Content-Type: application/json' \
  -d '{"resourceMetrics":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"manual-otel-smoke"}}]},"scopeMetrics":[{"metrics":[{"name":"codex.otel_smoke_metric","description":"OTLP smoke-test metric","unit":"1","sum":{"aggregationTemporality":2,"isMonotonic":true,"dataPoints":[{"asInt":"1","attributes":[{"key":"source","value":{"stringValue":"manual"}}]}]}}]}]}]}'
```

Collector側のPrometheus endpointを確認します。

```sh
curl http://localhost:9464/metrics
```

次のようなmetricが見えればOKです。

```text
codex_otel_smoke_metric_total 1
```

Prometheusからも確認します。

```sh
curl --get http://localhost:9090/api/v1/query \
  --data-urlencode 'query=codex_otel_smoke_metric_total'
```

ここで値が返れば、メトリクスの流れは通っています。

```text
curl -> Collector -> Prometheus -> Grafana
```

## 15. Grafanaで見る

ブラウザでGrafanaを開きます。

```text
http://localhost:3000
```

ログを見る場合:

1. 左メニューからExploreを開く
2. datasourceに `Loki` を選ぶ
3. 次のクエリを入れる

```logql
{service_name="codex-local-demo"}
```

スモークテストログだけなら:

```logql
{event_name="codex.otel_smoke_test"}
```

メトリクスを見る場合:

1. Exploreを開く
2. datasourceに `Prometheus` を選ぶ
3. 次のクエリを入れる

```promql
codex_otel_smoke_metric_total
```

ここまで見えたら、Grafana側の準備はOKです。

## 16. Codexを実際に動かす

次にCodex本体をOTel設定付きで起動します。

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

起動したら、軽い作業を頼みます。

```text
Say hello, then run `pwd`.
```

その後、Codexセッションを終了します。Codexはテレメトリを非同期にbatchして送るため、終了時にflushされることがあります。

## 17. Codexのログを見る

Grafana ExploreでLokiを選びます。

まずは全体を見ます。

```logql
{service_name="codex-local-demo"}
```

Codexっぽいイベントに絞ります。

```logql
{service_name="codex-local-demo"} |= "codex."
```

イベント名としては、Codexのバージョンや実行内容によって変わりますが、次のようなものが候補です。

```text
codex.conversation_starts
codex.api_request
codex.user_prompt
codex.tool_decision
codex.tool_result
```

`log_user_prompt = false` のため、プロンプト本文は出ない想定です。

## 18. Codexのメトリクスを見る

Grafana ExploreでPrometheusを選びます。

まずはservice labelで探します。

```promql
{service_name="codex-local-demo"}
```

metric名はOTelからPrometheusへ変換されます。

たとえば:

```text
codex.otel_smoke_metric -> codex_otel_smoke_metric_total
```

Codexのメトリクス候補:

```text
codex_api_request
codex_api_request_duration_ms
codex_tool_call
codex_tool_call_duration_ms
codex_turn_*
```

Codexのバージョンや実行経路によって出るmetricは変わる可能性があります。まずはExploreで `{service_name="codex-local-demo"}` から探すのが安全です。

## 19. トラブルシュートの考え方

うまく見えないときは、上流から順番に確認します。

```text
Codex
  -> Collector
    -> Loki / Prometheus
      -> Grafana
```

**Collectorに届いているか**

```sh
docker compose logs -f otel-collector
```

`debug` exporterを入れているので、受け取ったlogs/metricsがここに表示されます。

**Lokiがreadyか**

```sh
curl http://localhost:3100/ready
```

**PrometheusがCollectorを見られているか**

```sh
curl http://localhost:9090/api/v1/targets
```

**Grafanaの時間範囲が狭すぎないか**

Grafana右上の時間範囲を `Last 30 minutes` などにします。

**Codexがflushしているか**

Codexセッションを終了してからGrafanaを見ます。

## 20. Docker Desktopで詰まった話

今回、Docker Desktop起動時に次のようなエラーが出ました。

```text
Cannot resize ".../Docker.raw": permission denied
```

原因は、DockerのVMディスクファイル `Docker.raw` の所有者が `root` になっていたことでした。

確認すると、次のような状態でした。

```text
Docker.raw owner=root group=staff mode=-rw-r--r--
```

Docker Desktopは通常ユーザーで動くため、`root` 所有の `Docker.raw` をresizeできずに失敗していました。

直す場合は、Docker Desktopを終了した状態で所有者を戻します。

```sh
sudo chown kuma:staff /Users/kuma/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw
chmod u+rw /Users/kuma/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw
```

また、ディスク空き容量も重要です。空きが少ないとimage pullやVM resizeで詰まります。

```sh
df -h
docker system df
docker system prune
```

## 21. GitHubへ置く

最後にGitHubへpushしました。

```sh
git init
git add .
git commit -m "Add Codex OTel local demo"
git remote add origin git@github.com:kuma3/agent-usage-with-otel.git
git push -u origin main
```

今回のrepo:

```text
https://github.com/kuma3/agent-usage-with-otel
```

GitHub CLIを使う場合は、認証状態を確認します。

```sh
gh auth status
```

SSH remoteを使う場合は、SSH鍵がGitHubに登録されていて、ssh-agentに読み込まれている必要があります。

```sh
ssh -T git@github.com
ssh-add ~/.ssh/id_ed25519
```

## 22. ここまでで学べること

このハンズオンで触った内容は、かなり実践寄りです。

- OTelのlogsとmetricsの基本
- OpenTelemetry Collectorのreceiver / processor / exporter / pipeline
- Lokiでログを見る流れ
- Prometheusでメトリクスを見る流れ
- Grafana Exploreでの調査
- Docker Composeで観測基盤をローカルに立てる方法
- CodexのOTel出力設定

重要なのは、Grafanaそのものよりも「データがどこから来て、どこを通って、どこに保存され、どう表示されるか」です。

今回の構成を一言で言うと、次の通りです。

```text
Codexの中で起きたことを、OTel形式で外に出し、
Collectorで受け取り、
logsはLokiへ、metricsはPrometheusへ送り、
Grafanaで見る。
```

これが理解できれば、別のアプリケーションや別の監視基盤にも応用できます。

## 23. 次にやるなら

発展させるなら、次のような方向があります。

- tracesも有効化してTempoで見る
- dashboardをCodex用に作り込む
- prompt本文を送る場合のリスクとマスキング方針を整理する
- Collector processorで不要な属性を落とす
- ローカルだけでなくクラウドのOTel backendへ送る
- agent実行ごとのconversation idでログを絞り込む

まずは今回の構成を小さく理解しておくと、次の拡張がかなり楽になります。
