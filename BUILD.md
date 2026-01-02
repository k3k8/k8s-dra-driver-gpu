# Build and Deploy Guide for GPU Sharing Feature

This branch enables **GPU sharing** by setting `AllowMultipleAllocations: true` for GPU devices. This allows multiple Pods to share a single GPU using memory-based allocation.

## What Changed

Single line addition in `cmd/gpu-kubelet-plugin/deviceinfo.go`:

```go
// Enable GPU sharing by allowing multiple pod allocations per GPU
device.AllowMultipleAllocations = ptr.To(true)
```

## Build the Container Image

### Prerequisites
- Docker with buildx support
- Access to a container registry (Docker Hub, GHCR, etc.)

### Build Commands

#### 1. Build for Single Architecture (local testing)

```bash
# Build for current architecture (amd64 or arm64)
make -C deployments/container build \
  IMAGE_NAME=<your-registry>/<image-name> \
  IMAGE_TAG=<tag>

# Example:
make -C deployments/container build \
  IMAGE_NAME=ghcr.io/k3k8/k8s-dra-driver-gpu \
  IMAGE_TAG=gpu-sharing-v1
```

#### 2. Build Multi-Architecture Image (production)

```bash
# Build and push for both amd64 and arm64
make -C deployments/container build \
  BUILD_MULTI_ARCH_IMAGES=true \
  PUSH_ON_BUILD=true \
  IMAGE_NAME=<your-registry>/<image-name> \
  IMAGE_TAG=<tag>

# Example:
make -C deployments/container build \
  BUILD_MULTI_ARCH_IMAGES=true \
  PUSH_ON_BUILD=true \
  IMAGE_NAME=ghcr.io/k3k8/k8s-dra-driver-gpu \
  IMAGE_TAG=gpu-sharing-v1
```

#### 3. Push Image (if built without PUSH_ON_BUILD)

```bash
docker push <your-registry>/<image-name>:<tag>
```

## Deploy with Helm

### Install

```bash
helm install nvidia-dra-driver-gpu \
  ./deployments/helm/nvidia-dra-driver-gpu \
  --namespace nvidia-dra-driver-gpu \
  --create-namespace \
  --set image.repository=<your-registry>/<image-name> \
  --set image.tag=<tag> \
  --set resources.gpus.enabled=true
```

**Example:**
```bash
helm install nvidia-dra-driver-gpu \
  ./deployments/helm/nvidia-dra-driver-gpu \
  --namespace nvidia-dra-driver-gpu \
  --create-namespace \
  --set image.repository=ghcr.io/k3k8/k8s-dra-driver-gpu \
  --set image.tag=gpu-sharing-v1 \
  --set resources.gpus.enabled=true
```

### Upgrade

```bash
helm upgrade nvidia-dra-driver-gpu \
  ./deployments/helm/nvidia-dra-driver-gpu \
  --namespace nvidia-dra-driver-gpu \
  --set image.repository=<your-registry>/<image-name> \
  --set image.tag=<tag> \
  --set resources.gpus.enabled=true
```

### Uninstall

```bash
helm uninstall nvidia-dra-driver-gpu \
  --namespace nvidia-dra-driver-gpu
```

## Usage: Memory-Based GPU Sharing

Create `ResourceClaimTemplate` with memory requests:

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
              memory: "6Gi"  # Request 6GiB of GPU memory
          selectors:
          - cel:
              expression: "device.attributes['gpu.nvidia.com'].productName == 'NVIDIA GeForce RTX 3090'"
```

Then reference it in your Pod:

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

## Verification

Check that the driver is running:

```bash
kubectl get pods -n nvidia-dra-driver-gpu
```

Verify GPU devices are discovered:

```bash
kubectl get resourceslices
```

Check that `AllowMultipleAllocations` is enabled:

```bash
kubectl get resourceslice <name> -o yaml | grep -A5 allowMultipleAllocations
```

## Notes

- This implementation uses **memory-based allocation** instead of numeric allocation counts
- The existing `memory` capacity attribute in the DRA API is sufficient for GPU sharing
- No additional ConfigMaps or custom attributes are needed
- Each Pod requests the amount of GPU memory it needs (e.g., `memory: "6Gi"`)
- The scheduler ensures that the total allocated memory doesn't exceed the GPU's total memory
