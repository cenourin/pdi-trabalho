# Pipeline de T2 do colega (referência) — `feat/t2-ny-SIFT`

Fonte: repositório de terceiro em `Nova pasta/trabalho-PDI` (branch `origin/feat/t2-ny-SIFT`,
**não mergeada em `main`**, 9 commits `4032ffe..b1a4a3f`). Documento gerado por leitura do
código-fonte — não é um resumo do enunciado.

Não usa Docker/uv/CLI: aqui só interessam a lógica de reconhecimento e os parâmetros, que são
puro `scikit-image` + `numpy` e dá pra portar direto pra um notebook Jupyter local.

---

## 1. Decisão de arquitetura (ADR 0006)

- **Descritor: `skimage.feature.SIFT`** (padrão), `ORB` opcional via config. Sem OpenCV —
  `scikit-image` já traz SIFT/ORB e `match_descriptors`; o enunciado *permite* FLANN mas não
  *exige*; força bruta é suficiente na escala do dataset (≤ ~6k crops pequenos × 18 templates).
- **Matching:** `match_descriptors(cross_check=True, max_ratio=…)` — teste de razão de Lowe +
  cross-check, filtros de precisão.
- **Verificação geométrica:** RANSAC com `AffineTransform` sobre os matches válidos — só conta
  como match "de verdade" o que concorda com uma única transformação afim. Isso descarta
  matches espúrios em palavras que aparecem em todo rótulo (marca, "CONGELADO", tabela de
  peso), que é a principal fonte de falso positivo nesse tipo de matching.
- **Score:** `matches válidos (inliers) / nº keypoints do template`.
- **Decisão final:** o template com maior score vence **somente se** o score bater o limiar
  `min_match_frac`; caso contrário → `"unknown"`. Um mesmo frame pode gerar vários segmentos no
  T1 → o score da imagem é o **máximo** por template entre todos os segmentos dela.
- Templates ficam em `templates/<Classe>.png`, um por classe, committados. Bootstrapados
  automaticamente a partir do crop do detector do T1, mas substituíveis por crops manuais a
  qualquer momento (o reconhecedor só lê o diretório).
- `unknown` é tratado como saída válida e esperada — preferir "não sei" a errar a classe é o
  objetivo de design, alinhado com a regra do enunciado de que falso positivo pesa mais que
  detecção perdida.

---

## 2. Parâmetros (`RecognitionConfig`, valores padrão)

```python
descriptor: str = "sift"          # "sift" | "orb"
max_ratio: float = 0.72           # teste de razão de Lowe (match_descriptors)
cross_check: bool = True          # match_descriptors
min_match_frac: float = 0.05      # único knob de calibração exigido pelo enunciado
min_keypoints: int = 8            # abaixo disso, extract_features retorna None ("sem evidência")
orb_keypoints: int = 500          # só usado se descriptor="orb"
ransac_min_matches: int = 6       # mínimo de matches ratio+cross-check p/ tentar RANSAC
ransac_residual_px: float = 8.0   # threshold de resíduo do RANSAC (px)
```

`min_match_frac` é o único parâmetro pensado para ser calibrado por classe/dataset; os demais
são fixos e compartilhados entre todas as classes (sem tuning por classe).

Extração de template mínima: `MIN_TEMPLATE_KEYPOINTS = 60` (usado só no script de bootstrap de
templates, não no matching em si).

---

## 3. Pipeline passo a passo

1. **`extract_features(imagem, config)`** (`features.py`)
   - `SIFT()` ou `ORB(n_keypoints=orb_keypoints)` do `skimage.feature`.
   - `extractor.detect_and_extract(image)`; captura `RuntimeError`/`ValueError` (skimage
     lança quando não há keypoints/oitavas suficientes) → retorna `None`.
   - Se `descriptors.shape[0] < min_keypoints` → retorna `None`.
   - Retorna `Features(keypoints, descriptors)` (keypoints em coordenadas `(row, col)`).

2. **`match_fraction(template, segmento, config)`** (`matching.py`)
   - `match_descriptors(template.descriptors, segmento.descriptors, cross_check=True, max_ratio=0.72)`.
   - Se `len(matches) < ransac_min_matches` (6) → score = 0.0.
   - `ransac((src, dst), AffineTransform, min_samples=3, residual_threshold=8.0, max_trials=200, rng=0)`.
   - Se `ValueError` ou `inliers is None` → score = 0.0.
   - `score = count_nonzero(inliers) / len(template)` (proporção sobre o nº de keypoints do
     **template**, não do segmento).

3. **`classify_features` / `classify_segment`** (`classify.py`)
   - Roda `match_fraction` do segmento contra **todos** os templates.
   - Escolhe o template com maior score.
   - Se `score < min_match_frac` → `Prediction(label="unknown", score=score, scores={...})`.
   - Senão → `Prediction(label=melhor_template, score=score, scores={...})`.

4. **Agregação em lote** (`batch.py`, usado no CLI `pdiseg-recognize`)
   - `find_segment_crops`: varre `<segments_root>/<Classe>/<stem>_segment{ed,ada}_<N>.png`
     (regex `_(?:segmented|segmentada)_\d+$`).
   - `score_dataset`: extrai features e roda `match_fraction` contra cada template, para cada
     crop (thread pool opcional, como no `calibrate`).
   - `aggregate_by_image`: agrupa por `(classe, stem_da_imagem_fonte)`, pega o **máximo** score
     por template entre os segmentos da mesma imagem.
   - `predict_images`: aplica o gate `min_match_frac` sobre os scores agregados por imagem.
   - `summarize`: `accuracy`, `unknown`, `false_positives`, `false_positive_rate`.
   - `sweep_thresholds`: reaplica `min_match_frac` de 0.00 a 0.50 (passo 0.01) sobre os scores
     já calculados, sem re-extrair features — é assim que ele varre o parâmetro sem custo extra
     de reprocessamento.

---

## 4. Resultado da varredura de calibração (900 imagens, 7/18 templates)

| `min_match_frac` | accuracy | unknown | falsos positivos | FP rate |
|---|---:|---:|---:|---:|
| 0.00 | 31.7% | 0   | 615 | 68.3% |
| 0.01 | 31.7% | 232 | 383 | 42.6% |
| **0.05 (padrão)** | **27.6%** | 566 | 86 | **9.6%** |
| 0.10 | 14.0% | 770 | 4 | 0.4% |
| 0.13 | 6.8%  | 839 | 0 | 0.0% |

Escolheu 0.05 como padrão: mantém FP abaixo de 10% preservando recall, dado que só 7 das 18
classes têm template. Accuracy é limitada pela cobertura de templates, não pelo matcher — a
alavanca principal seria completar os templates das 11 classes faltando.

---

## 5. Lacuna conhecida (documentada por ele mesmo)

> Só 7 das 18 classes do dataset têm template committado. Segmentos das outras 11 só podem dar
> `unknown` ou falso positivo, o que limita o teto de accuracy possível.

---

## 6. Estrutura de código (para referência, não para copiar literalmente)

```
src/pdiseg/recognition/
├── config.py      # RecognitionConfig (dataclass frozen)
├── features.py    # extract_features / extract_descriptors (skimage SIFT/ORB)
├── matching.py     # match_fraction (ratio + cross-check + RANSAC affine)
├── classify.py     # load_templates, classify_features, classify_segment
└── batch.py         # score_dataset, aggregate_by_image, predict_images, sweep_thresholds, CSV writers
src/pdiseg/cli/recognize.py    # CLI pdiseg-recognize (wrapper fino sobre batch.py)
scripts/build-templates.py     # bootstrap de templates/<Classe>.png a partir do crop do T1
```

Módulos-chave a olhar se for portar a lógica para um notebook: `features.py` (extração),
`matching.py` (a função `match_fraction` é o coração do método), `classify.py` (regra de
decisão + `unknown`).

---

## 7. Snippet mínimo equivalente (para portar em notebook local, sem o resto do repo)

```python
import numpy as np
from skimage.feature import SIFT, match_descriptors
from skimage.measure import ransac
from skimage.transform import AffineTransform

def extract(img_gray, min_keypoints=8):
    sift = SIFT()
    try:
        sift.detect_and_extract(img_gray)
    except (RuntimeError, ValueError):
        return None
    kp, desc = sift.keypoints, sift.descriptors
    if desc is None or desc.shape[0] < min_keypoints:
        return None
    return kp, desc

def match_fraction(template, segmento, max_ratio=0.72, ransac_min_matches=6, residual_px=8.0):
    kp_t, desc_t = template
    kp_s, desc_s = segmento
    matches = match_descriptors(desc_t, desc_s, cross_check=True, max_ratio=max_ratio)
    if len(matches) < ransac_min_matches:
        return 0.0
    src, dst = kp_t[matches[:, 0]], kp_s[matches[:, 1]]
    try:
        _, inliers = ransac((src, dst), AffineTransform, min_samples=3,
                             residual_threshold=residual_px, max_trials=200, rng=0)
    except ValueError:
        return 0.0
    if inliers is None:
        return 0.0
    return np.count_nonzero(inliers) / len(kp_t)

# decisão: melhor template vence se score >= min_match_frac (0.05), senão "unknown"
```

---

## 8. Reprodução no repo original (para inspecionar, não necessário para nós)

```sh
make templates      # bootstrap templates/ a partir do detector do T1
make run             # gera segmentos T1 em result/
make recognize       # -> result/recognition.csv + calibration/recognition_sweep.csv
make report           # rebuild dos PDFs em docs/report/
```

Docs originais: `docs/adr/0006-local-feature-recognition.md`, `UPDATES.md`,
`docs/report/t2_report.pdf`, `docs/report/t2_simplified.pdf` (na branch `feat/t2-ny-SIFT`).
