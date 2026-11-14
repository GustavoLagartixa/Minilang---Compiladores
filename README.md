# MiniComp: front-end LL(1) para a MiniLang

[![Abrir no Google Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1Z4JWASsHAYpJIiW2OWqv_-QG1GJyYrpX)

Entrega do front-end da MiniLang para a disciplina de Compiladores. O projeto tem o analisador léxico, um parser LL(1) preditivo dirigido por tabela (com pilha explícita, sem descida recursiva), a geração da AST, a tabela de símbolos e um interpretador que executa os programas a partir da AST. Está tudo em C++17, sem bibliotecas externas.

**Grupo:** Arthur Batista Correa Alves, Davi Sousa Alves, Felipe Francisco Ferreira de Lima, Guilherme Augusto Boquimpani e Gustavo Santiago de Almeida.

## Como rodar

### No Google Colab (mais rápido)

Para quem não quiser instalar nada, deixamos um notebook pronto: [abrir no Colab](https://colab.research.google.com/drive/1Z4JWASsHAYpJIiW2OWqv_-QG1GJyYrpX). Ele precisa de uma conta Google. Basta usar *Ambiente de execução > Executar tudo*, ou rodar as células em ordem. O notebook clona o repositório, compila com `make`, roda os testes e executa os três exemplos da especificação.

Se preferir montar um notebook do zero, são duas células:

```
!git clone <URL-DESTE-REPOSITORIO> minilang
%cd minilang
!make
!make test
```
```
!echo 5 | ./mlc --run tests/valid/fatorial.ml
!echo "7 3" | ./mlc --run tests/valid/maximo.ml
!echo "5 3 2 0" | ./mlc --run tests/valid/soma_ate_zero.ml
!./mlc tests/invalid/spec_5_4.ml
```

O `make test` deve terminar com `passaram: 17, falharam: 0`. Os três exemplos imprimem `120`, `7` e `10`, e o último comando mostra os erros sintáticos do exemplo inválido da seção 5.4 da especificação.

### Na própria máquina

Precisa de `g++` com C++17 (testamos com GCC 8 em diante) e `make`. Em Linux ou WSL:

1. Clone o repositório e entre na pasta.
2. Rode `make`. Isso gera o executável `./mlc`.
3. Rode `make test` para executar os 17 casos de teste.
4. Execute um programa: `echo 5 | ./mlc --run tests/valid/fatorial.ml`. A entrada do `read` vem da entrada padrão, com os valores separados por espaço.

No Windows sem WSL, usamos o MSYS2 (UCRT64) e compilamos direto, já que não há `make` por padrão:

```
g++ -std=c++17 -O2 -o mlc.exe src/main.cpp src/lexer.cpp src/grammar.cpp src/parser.cpp src/ast.cpp src/interp.cpp
```

Se os acentos saírem quebrados no terminal do Windows, `chcp 65001` resolve.

### Opções do `mlc`

Sem opções, o programa só valida o arquivo (léxico e sintático). As demais opções são:

- `--tokens` lista os tokens com linha e coluna;
- `--symbols` mostra a tabela de símbolos;
- `--ast` imprime a árvore sintática abstrata;
- `--run` executa o programa, lendo a entrada pela stdin;
- `--grammar` imprime as produções, FIRST, FOLLOW e a tabela LL(1) (não precisa de arquivo).

O código de saída é 0 quando tudo vai bem, 1 em erro léxico ou sintático, 2 em erro de execução e 3 em uso incorreto.

## Organização

O código está em `src/`, com um par de arquivos por etapa: `lexer` (scanner), `grammar` (produções, FIRST/FOLLOW e construção da tabela, com detecção de conflitos), `parser` (a pilha, a árvore de derivação e a recuperação de erros), `ast` (construção da AST a partir da árvore de derivação) e `interp` (interpretador). O `main.cpp` só cuida das opções de linha de comando.

Em `docs/grammar.md` estão FIRST, FOLLOW e a tabela LL(1). Esse arquivo é gerado pelo próprio programa (`./mlc --grammar > docs/grammar.md`), então não precisa ser mantido à mão e não fica fora de sincronia com o código.

Os autômatos do JFLAP estão em `jflap/`: `id`, `number`, `keywords`, `assign_eq`, `relops`, `symbols` e `comment`. Eles abrem no JFLAP 7.1 e podem ser testados em *Input > Multiple Run*. Não há autômato para espaços em branco, e o de comentário não aceita espaço nem `[` dentro do comentário. São limitações do JFLAP, que lê espaço e tab como transição λ e dá erro interno com `[`. O scanner em C++ trata esses casos normalmente. O script `tools/gen_jflap.py` gera os arquivos e já os simula antes de gravar.

Os testes ficam em `tests/`: `valid/` tem programas com entrada (`.in`) e saída esperada (`.out`), `invalid/` tem programas com erro léxico ou sintático e `runtime/` tem erros de execução. Nos dois últimos, o arquivo `.expect` guarda um trecho da mensagem esperada.

## Decisões de projeto

A especificação tem algumas inconsistências, e escolhemos o seguinte:

**Corpo de `then`, `else` e `do` sem `begin/end`.** A regra 4 da seção 6.1 exige blocos, mas o exemplo `maximo` (seção 5.2) escreve `then write(a) else write(b)`. Como os exemplos precisam ser aceitos, o parser aceita as duas formas, com `<corpo> → <bloco> | <comando_ne>`.

**Else pendente.** Essa extensão introduz o conflito clássico em `<condicional'>` com `else`, porque `else` pertence a FOLLOW(`<condicional'>`). Resolvemos casando o `else` com o `if` mais próximo, e o conflito aparece como "resolvido" no `docs/grammar.md`. Fora esse caso, não há conflitos LL(1).

**Identificadores com `_`.** O exemplo `soma_ate_zero` usa sublinhado, que a regex da seção 3 (`[a-z][a-z0-9]*`) não permite. Usamos `[a-z][a-z0-9_]*`.

**Associatividade.** A gramática sem recursão à esquerda é recursiva à direita, e lida literalmente faria `10-3-2` valer `10-(3-2)`. Ao montar a AST, reagrupamos os operadores com associatividade à esquerda. Assim `10-3-2` dá 5 e `8/2*2` dá 8 (há um teste para isso em `tests/valid/associatividade.ml`).

**Comando vazio e ponto e vírgula.** Modelamos `<linhas>` como comandos separados por `;`, com comando vazio permitido. Isso cobre o último comando do bloco sem `;`, `begin end` e `;` sobrando antes do `end`.

**Variáveis.** A seção 6.3 limita a memória a 26 variáveis, mas os exemplos usam nomes como `fat` e `valor`. A tabela é indexada pelo nome da variável, e as não inicializadas valem 0.

**Faixa de valores.** Os inteiros vão de 0 a 2³¹−1. Resultado negativo ou estouro, divisão por zero, `read` sem entrada disponível e laço sem fim (limite de 50 milhões de passos) são erros de execução, com linha e coluna.

**Tokens fora da gramática.** `int`, `{`, `}` e `,` são reconhecidos pelo scanner, como diz a tabela de tokens, mas não aparecem nas produções. Se surgirem num programa, o parser reporta erro sintático.

## Tratamento de erros

O scanner não para no primeiro erro léxico: ele reporta todos (caractere inválido, maiúscula em identificador, número começando com dígito, número fora da faixa) e segue. No parser usamos recuperação em modo pânico: depois de um erro, descartamos tokens até `;`, `end`, `.` ou o fim do arquivo, desempilhamos até o topo da pilha conseguir tratar o token atual e continuamos. Dá para ver vários erros por execução, até 30. Um efeito conhecido é o erro em cascata. No exemplo 5.4, a falta do `then` gera também erros na linha seguinte. Quando há qualquer erro, a AST não é gerada.
