# PDI — Segmentação e Reconhecimento de Rótulos em Embalagens de Frango

Trabalho prático da disciplina de Processamento Digital de Imagens (PDI), dividido em duas
etapas sobre o mesmo conjunto de fotos de embalagens de produtos avícolas, capturadas em uma
esteira com câmera fixa:

- **T1 — Segmentação**: localizar e recortar a região do rótulo com o nome do produto em cada
  imagem, usando exclusivamente técnicas clássicas de PDI (espaços de cor, limiarização,
  morfologia, filtragem espacial, transformações geométricas). Falso positivo (recorte sem texto
  de produto, ou com o texto errado) pesa mais que detecção perdida.
- **T2 — Reconhecimento**: dado um template recortado manualmente por classe, identificar a qual
  produto pertence um segmento gerado no T1, usando descritores locais (SIFT/ORB) e casamento de
  descritores (FLANN), sem uso de IA/aprendizado de máquina para a classificação.

Autoria: **Bruno Silveira** ([@cenourin](https://github.com/cenourin)). O T2 foi desenvolvido em
dupla com **Monique Machado**, cuja contribuição está creditada em
[`docs/relatorio-t2.tex`](docs/relatorio-t2.tex).

## Estrutura do repositório

```
segmentador_colab.ipynb   pipeline T1 para embalagens tipo bandeja
segmentador_selado.ipynb  pipeline T1 para embalagens seladas
enunciado2026.pdf         especificação do T1

t2/
  enunciado-t2.md          especificação do T2 (transcrição do PDF original)
  enunciado-t2.pdf
  reconhecedor_t2.ipynb    pipeline de reconhecimento (SIFT/FLANN + RANSAC)
  templates-gold/          template gold por classe (1 imagem/classe, recorte manual)
  resultados_t2.csv        resultado bruto por imagem avaliada
  pipeline-colega-t2.md    notas de leitura sobre a abordagem de um colega (referência externa,
                           código de terceiro não incluído neste repositório)

docs/
  relatorio-tecnico.tex    relatório técnico completo (arquitetura, requisitos, dados, testes,
                           resultados, aplicação em caso real)
  relatorio-t2.tex / .pdf  relatório de resultados do T2 (entregável da disciplina)
  matriz_confusao_t2.png   matriz de confusão do T2
  resultados-exemplo/      1 imagem de resultado por classe do T1 (bandeja/ e selado/)
```

O dataset completo (`Train_and_Validation/`), as saídas completas dos pipelines
(`resultado_bandeja/`, `resultado_selado/`) e material de apoio do T2 (crops brutos, cópia de
repositório de terceiro usado como referência, pacote de entrega) **não são versionados** — ver
`.gitignore`. Apenas uma amostra representativa (1 resultado por classe no T1, os templates gold
no T2) fica no repositório.

## Resultados preliminares

Resumo visual dos resultados — os relatórios completos ficam em `docs/relatorio-t1.pdf` e
`docs/relatorio-t2.pdf`, com introdução, objetivos, desenvolvimento e resultados detalhados por
classe; esta seção existe pra dar o essencial sem precisar abrir os PDFs.

### T1 — Segmentação

Pipeline testado sobre as 18 classes do dataset, nos dois tipos de embalagem. Abaixo, um exemplo
de cada tipo — os 18 recortes (um por classe) estão em `docs/resultados-exemplo/` e o relatório
completo, com todas as classes, em [`docs/relatorio-t1.pdf`](docs/relatorio-t1.pdf).

<p align="center">
  <img src="docs/resultados-exemplo/bandeja/93000005_Meio_das_Asas_Congelado_2_crops.png" width="45%" alt="Recorte do rótulo Meio das Asas Congelado, embalagem tipo bandeja">
  <img src="docs/resultados-exemplo/selado/93000019_Meio_das_Asas_Congela_2_recortes.png" width="45%" alt="Recorte do rótulo Meio das Asas Congelado Selado, embalagem selada">
</p>
<p align="center"><em>Esquerda: Meio das Asas Congelado (bandeja). Direita: Meio das Asas Congelado Selado (selado).</em></p>

### T2 — Reconhecimento

Avaliação sobre 350 imagens (7 classes bandeja × 50 imagens cada), com `min_match_frac = 0,08`:

| Métrica | Valor |
|---|---|
| Acurácia | 83,4% (292/350) |
| Não identificado (`unknown`) | 13,7% (48/350) |
| Falsos positivos | 2,9% (10/350) |

Classes com texto curto e exclusivo (Coração, Filé de Peito) tiveram 100% de acerto; classes com
rótulo de menor contraste ou mais elementos repetidos entre embalagens (Filé de Coxas e
Sobrecoxas com Pele, Coxinhas das Asas) concentraram a maior parte dos segmentos não
identificados. Resultado completo por classe, templates utilizados e histórico de calibração em
[`docs/relatorio-t2.pdf`](docs/relatorio-t2.pdf).

<p align="center">
  <img src="docs/matriz_confusao_t2.png" width="70%" alt="Matriz de confusão do reconhecimento T2">
</p>
<p align="center"><em>Matriz de confusão por classe (min_match_frac = 0,08).</em></p>

## Como rodar

Dependências: `opencv-python`, `numpy`, `matplotlib` (instaladas via `%pip install` dentro dos
próprios notebooks).

1. Coloque o dataset em `Train_and_Validation/` na raiz do repositório (sibling dos notebooks).
2. Abra `segmentador_colab.ipynb` (bandeja) ou `segmentador_selado.ipynb` (selado) no
   Jupyter/Colab e rode todas as células — a última célula de cada notebook executa o pipeline
   completo no dataset e grava recortes + grids visuais em `resultado_bandeja/` ou
   `resultado_selado/`.
3. Para o T2, gere os segmentos do T1, corrija manualmente os que falharem, e rode
   `t2/reconhecedor_t2.ipynb` apontando para os segmentos e para `t2/templates-gold/`.

Nenhum parâmetro deve ser ajustado por imagem — ambos os pipelines rodam fim a fim sobre um
conjunto de imagens novo sem alteração de código, conforme exigido pela disciplina.

## Direitos autorais

As imagens do dataset e as marcas/produtos nelas retratados (embalagens da marca **Super
Frango**) pertencem aos seus respectivos titulares de direitos e foram cedidas apenas para uso
acadêmico nesta disciplina. Este repositório versiona apenas o **código de processamento de
imagem** desenvolvido pelo autor; nenhuma reivindicação de direito é feita sobre a marca, o
produto ou as imagens originais do dataset.
