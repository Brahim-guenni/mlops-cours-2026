# Benchmark de latence — german-credit, modèle v3

1000 appels par backend, après 10 appels de chauffe · x86_64 · Python 3.12.16

| Backend | P50 (ms) | P95 (ms) | P99 (ms) |
|---|---|---|---|
| sklearn_pipeline | 4.531 | 8.085 | 12.421 |
| onnx_python | 0.079 | 0.147 | 0.248 |
| onnx_batch_100 | 0.291 | 0.427 | 0.592 |

Accélération ONNX vs scikit-learn (P50, 1 ligne) : x57.2
