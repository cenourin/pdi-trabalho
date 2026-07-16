# Trabalho Prático 2 – Reconhecimento de Produtos por Correspondência de Características Locais

## Objetivo

Nesta segunda etapa do projeto, o objetivo é desenvolver um algoritmo capaz de identificar automaticamente o produto presente em uma imagem utilizando **descritores locais de características**, sem o emprego de técnicas de Inteligência Artificial ou Aprendizado de Máquina.

O reconhecimento deverá ser realizado por meio de **SIFT** ou **ORB** (ou outro descritor local), utilizando uma pequena região da embalagem como imagem de referência (*template*).

## Descrição da Atividade

Para cada uma das classes de produtos cuja embalagem é do tipo **bandeja**, existe um conjunto contendo aproximadamente **50 imagens**.

O aluno deverá selecionar **uma única imagem** de cada classe e, utilizando um editor de imagens, realizar **manualmente** o recorte de uma região característica da embalagem que apresente elevado potencial discriminativo.

Essa região será utilizada como **imagem pivô (template)** para representar toda a classe.

Após a construção do conjunto de templates, o aluno deverá desenvolver um algoritmo baseado em descritores locais para classificar uma imagem, consistindo de:

- Comparar cada SEGMENTO de imagem, obtido no trabalho 1, com todos os templates.
- Detectar pontos de interesse.
- Extrair descritores locais.
- Realizar o casamento (*matching*) entre descritores.
- Determinar qual template apresenta a melhor correspondência.
- Informar o produto identificado.

**ATENÇÃO !!!** Você deve corrigir manualmente a saída do seu trabalho T1. Ou seja, cada imagem deve possuir um segmento correto. Você pode simplesmente recortar manualmente a imagem.

A decisão deverá ser baseada exclusivamente na análise dos descritores locais.

Um dos principais objetivos deste trabalho consiste na calibração dos parâmetros do algoritmo de correspondência, que neste caso será apenas a **porcentagem de correspondências válidas** entre os pontos de interesse do template com os pontos de interesse do segmento comparado.

Espera-se que a solução minimize a ocorrência de **falsos positivos**, mesmo que isso resulte em uma redução da quantidade de correspondências detectadas.

## É permitido

- SIFT.
- ORB.
- FAST + BRIEF.
- FLANN (opencv)

## Entregáveis

Cada grupo deverá entregar:

1. Código-fonte completo.
2. Conjunto de templates utilizados (um para cada classe).
3. Relatório técnico em formato PDF contendo os resultados.
