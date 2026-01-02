# GPU共有機能のビルド＆デプロイガイド

このブランチは、GPUデバイスに `AllowMultipleAllocations: true` を設定することで **GPU共有** を有効化します。これにより、複数のPodがメモリベースの割り当てを使用して1つのGPUを共有できるようになります。

## 変更内容

`cmd/gpu-kubelet-plugin/deviceinfo.go` に1行追加しただけです：

```go
// Enable GPU sharing by allowing multiple pod allocations per GPU
device.AllowMultipleAllocations = ptr.To(true)
```

## コンテナイメージのビルド

### 前提条件
- Docker（buildxサポート必須）
- コンテナレジストリへのアクセス（Docker Hub、GHCR等）

### ビルドコマンド

#### 1. シングルアーキテクチャビルド（ローカルテスト用）

```bash
# 現在のアーキテクチャ（amd64またはarm64）用にビルド
make -C deployments/container build \
  IMAGE_NAME=<your-registry>/<image-name> \
  IMAGE_TAG=<tag>

# 例：
make -C deployments/container build \
  IMAGE_NAME=ghcr.io/k3k8/k8s-dra-driver-gpu \
  IMAGE_TAG=gpu-sharing-v1
```

#### 2. マルチアーキテクチャビルド（本番環境用）

```bash
# amd64とarm64の両方をビルドしてプッシュ
make -C deployments/container build \
  BUILD_MULTI_ARCH_IMAGES=true \
  PUSH_ON_BUILD=true \
  IMAGE_NAME=<your-registry>/<image-name> \
  IMAGE_TAG=<tag>

# 例：
make -C deployments/container build \
  BUILD_MULTI_ARCH_IMAGES=true \
  PUSH_ON_BUILD=true \
  IMAGE_NAME=ghcr.io/k3k8/k8s-dra-driver-gpu \
  IMAGE_TAG=gpu-sharing-v1
```

#### 3. イメージのプッシュ（PUSH_ON_BUILDなしでビルドした場合）

```bash
docker push <your-registry>/<image-name>:<tag>
```

## Helmでデプロイ

### インストール

```bash
helm install nvidia-dra-driver-gpu \
  ./deployments/helm/nvidia-dra-driver-gpu \
  --namespace nvidia-dra-driver-gpu \
  --create-namespace \
  --set image.repository=<your-registry>/<image-name> \
  --set image.tag=<tag> \
  --set resources.gpus.enabled=true
```

**例：**
```bash
helm install nvidia-dra-driver-gpu \
  ./deployments/helm/nvidia-dra-driver-gpu \
  --namespace nvidia-dra-driver-gpu \
  --create-namespace \
  --set image.repository=ghcr.io/k3k8/k8s-dra-driver-gpu \
  --set image.tag=gpu-sharing-v1 \
  --set resources.gpus.enabled=true
```

### アップグレード

```bash
helm upgrade nvidia-dra-driver-gpu \
  ./deployments/helm/nvidia-dra-driver-gpu \
  --namespace nvidia-dra-driver-gpu \
  --set image.repository=<your-registry>/<image-name> \
  --set image.tag=<tag> \
  --set resources.gpus.enabled=true
```

### アンインストール

```bash
helm uninstall nvidia-dra-driver-gpu \
  --namespace nvidia-dra-driver-gpu
```

## 使用方法：メモリベースのGPU共有

メモリリクエストを含む `ResourceClaimTemplate` を作成：

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-6gi-3090
  namespace: coder
spec:
  spec:
    devices:
      requests:
      - name: "gpu"
        exactly:
          deviceClassName: "gpu.nvidia.com"
          capacity:
            requests:
              memory: "6Gi"  # 6GiBのGPUメモリをリクエスト
          selectors:
          - cel:
              expression: "device.attributes['gpu.nvidia.com'].productName == 'NVIDIA GeForce RTX 3090'"
```

Podで参照：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-gpu-pod
spec:
  resourceClaims:
  - name: gpu-claim
    resourceClaimTemplateName: gpu-6gi-3090
  containers:
  - name: cuda-container
    image: nvidia/cuda:12.0.0-base-ubuntu22.04
    command: ["nvidia-smi"]
    resources:
      claims:
      - name: gpu-claim
```

## 動作確認

ドライバーが稼働していることを確認：

```bash
kubectl get pods -n nvidia-dra-driver-gpu
```

GPUデバイスが検出されていることを確認：

```bash
kubectl get resourceslices
```

`AllowMultipleAllocations` が有効化されていることを確認：

```bash
kubectl get resourceslice <name> -o yaml | grep -A5 allowMultipleAllocations
```

## 補足

- この実装は**数値ベースの割り当て**ではなく、**メモリベースの割り当て**を使用します
- DRA APIに既存の `memory` Capacity属性だけでGPU共有が実現できます
- 追加のConfigMapやカスタム属性は不要です
- 各Podは必要なGPUメモリ量をリクエストします（例: `memory: "6Gi"`）
- スケジューラは、割り当てられたメモリの合計がGPUの総メモリを超えないようにします

## 実際の運用例

RTX 3090（24GB）を複数Podで共有する例：

### パターン1: 4分割（6GB × 4）

```yaml
# gpu-6gi-3090.yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-6gi-3090
spec:
  spec:
    devices:
      requests:
      - name: "gpu"
        exactly:
          deviceClassName: "gpu.nvidia.com"
          capacity:
            requests:
              memory: "6Gi"
          selectors:
          - cel:
              expression: "device.attributes['gpu.nvidia.com'].productName == 'NVIDIA GeForce RTX 3090'"
```

このテンプレートを使うと、1台の RTX 3090 に最大4つのPodを割り当てられます（6GB × 4 = 24GB）。

### パターン2: 専有利用（24GB × 1）

```yaml
# gpu-24gi-3090.yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-24gi-3090
spec:
  spec:
    devices:
      requests:
      - name: "gpu"
        exactly:
          deviceClassName: "gpu.nvidia.com"
          capacity:
            requests:
              memory: "24Gi"  # フル容量をリクエスト
          selectors:
          - cel:
              expression: "device.attributes['gpu.nvidia.com'].productName == 'NVIDIA GeForce RTX 3090'"
```

このテンプレートを使うと、GPU全体を1つのPodが専有します。

### パターン3: 柔軟な共有（any GPU model）

```yaml
# gpu-6gi-any.yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-6gi-any
spec:
  spec:
    devices:
      requests:
      - name: "gpu"
        exactly:
          deviceClassName: "gpu.nvidia.com"
          capacity:
            requests:
              memory: "6Gi"
          # セレクタなし = どのGPUモデルでもOK
```

このテンプレートを使うと、どのGPUモデルでも6GBのメモリが確保されます。

## トラブルシューティング

### Podがスケジュールされない場合

```bash
# ResourceClaimの状態を確認
kubectl describe resourceclaim <claim-name>

# ResourceSliceを確認
kubectl get resourceslices -o yaml | grep -A10 "memory"

# ドライバーのログを確認
kubectl logs -n nvidia-dra-driver-gpu -l app.kubernetes.io/name=nvidia-dra-driver-gpu
```

### メモリ不足でスケジュールできない場合

利用可能なGPUメモリより少ない量をリクエストしてください：

```yaml
# ❌ 間違い: RTX 3060（12GB）に24GBをリクエスト
memory: "24Gi"

# ✅ 正しい: RTX 3060に12GB以下をリクエスト
memory: "12Gi"
```

### 複数Podが同じGPUに割り当てられない場合

1. `AllowMultipleAllocations: true` が設定されているか確認：
   ```bash
   kubectl get resourceslice <name> -o yaml | grep allowMultipleAllocations
   ```

2. 合計メモリ量がGPUの容量を超えていないか確認：
   ```bash
   # 例: 6GB × 5 = 30GB > 24GB（RTX 3090） → 5つ目のPodは割り当てられない
   ```
