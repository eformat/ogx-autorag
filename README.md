# OGX AutoRAG Stack

Deploys the full infrastructure needed to run AutoRAG on Red Hat OpenShift AI 3.4:

- **Milvus** (standalone + etcd) -- remote vector database for RAG indexing
- **LlamaStack** -- inference server with bge-m3 embedding and foundation models (qwen35-9b, llama-4-scout) via MaaS
- **MinIO** -- S3-compatible object storage for pipeline artifacts and documents
- **Data Science Pipelines (DSPA)** -- pipeline server required by AutoRAG
- **Connection secrets** -- LlamaStack and S3 connections pre-configured for the AutoRAG UI

## Prerequisites

- OpenShift cluster with RHOAI 3.4+ installed
- `LlamaStackDistribution` CRD available (LlamaStack operator enabled)
- Target namespace exists and user has `admin` role
- MaaS endpoint accessible at `maas.apps.ocp.cloud.rhai-tmm.dev`

## Deploy to a namespace

1. Create an overlay for the target namespace:

```bash
NS=user-yourname
mkdir -p overlays/$NS

cat > overlays/$NS/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: $NS
resources:
  - ../../base
patches:
  - target:
      kind: DataSciencePipelinesApplication
      name: dspa
    patch: |
      - op: replace
        path: /spec/objectStorage/externalStorage/host
        value: minio-pipeline.$NS.svc.cluster.local:9000
  - target:
      kind: Secret
      name: llama-stack-connection
    patch: |
      - op: replace
        path: /stringData/LLAMA_STACK_CLIENT_BASE_URL
        value: "http://lsd-genai-playground-service.$NS.svc.cluster.local:8321"
      - op: replace
        path: /stringData/OGX_CLIENT_BASE_URL
        value: "http://lsd-genai-playground-service.$NS.svc.cluster.local:8321"
EOF
```

2. Deploy (optionally impersonate the namespace owner):

```bash
oc kustomize overlays/$NS | oc apply --as=owner@redhat.com -f -
```

3. Wait for everything to come up:

```bash
oc get llsd -n $NS                          # Phase: Ready
oc get dspa -n $NS -o jsonpath='{.items[0].status.conditions[?(@.type=="Ready")].status}'  # True
```

## RHOAI 3.4.0 pipeline image fix

RHOAI 3.4.0 ships with a hardcoded image SHA that doesn't exist in the registry. Before creating an AutoRAG run, import the fixed pipeline:

1. Download the fixed pipeline YAML (already in `pipeline-fix/pipeline.yaml`, or from [red-hat-data-services/pipelines-components](https://github.com/red-hat-data-services/pipelines-components/blob/rhoai-3.4-fixed/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.yaml))

2. In the RHOAI dashboard: **Develop & train > Pipelines > Pipeline Definitions > Import Pipeline**
   - Name: `documents-rag-optimization-pipeline`
   - Upload: `pipeline-fix/pipeline.yaml`

3. Or via CLI:

```bash
DS_ROUTE=$(oc get route ds-pipeline-dspa -n $NS -o jsonpath='{.spec.host}')
curl -sk -X POST "https://$DS_ROUTE/apis/v2beta1/pipelines/upload?name=documents-rag-optimization-pipeline" \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -F "uploadfile=@pipeline-fix/pipeline.yaml"
```

## Create an AutoRAG run

1. In the RHOAI dashboard: **Gen AI studio > AutoRAG > Create AutoRAG optimization run**
2. Select the **LlamaStack AutoRAG** connection
3. Select **MinIO Pipeline** as the S3 connection
4. Upload documents and an evaluation dataset (see `test-data/` for examples)
5. Select **milvus (remote Milvus)** as the Vector I/O provider
6. Click **Create run**

## Repository structure

```
base/
  milvus/          # Milvus standalone + etcd
  llama-stack/     # LlamaStackDistribution + config + connection secrets
  pipeline/        # MinIO + DSPA + S3 connection
overlays/
  user-<name>/     # Per-namespace kustomize overlay
pipeline-fix/      # Fixed pipeline YAML for RHOAI 3.4.0
test-data/         # Sample PDF and evaluation dataset
```

## Models

| Model | Type | Provider | Source |
|-------|------|----------|--------|
| bge-m3 | Embedding (1024 dims) | remote::vllm | MaaS |
| qwen35-9b | Foundation LLM | remote::vllm | MaaS |
| llama-4-scout-17b | Foundation LLM | remote::vllm | MaaS |
