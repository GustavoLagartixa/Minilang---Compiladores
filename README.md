# MiniComp: Front-End LL(1) para a linguagem MiniLang

[![Abrir no Google Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1Z4JWASsHAYpJIiW2OWqv_-QG1GJyYrpX)

Projeto da disciplina de Compiladores: analisador léxico, parser LL(1) preditivo dirigido por tabela
(pilha explícita), árvore sintática abstrata (AST), tabela de símbolos e interpretador da linguagem
didática MiniLang, em C++17, sem dependências externas.

**Integrantes:** Arthur Batista Correa Alves · Davi Sousa Alves · Felipe Francisco Ferreira de Lima ·
Guilherme Augusto Boquimpani · Gustavo Santiago de Almeida

## Executar sem instalar nada (Google Colab)

O jeito mais rápido de avaliar o projeto é o notebook do Colab, que já tem `g++` e `make`:

**[Abrir o notebook no Google Colab](https://colab.research.google.com/drive/1Z4JWASsHAYpJIiW2OWqv_-QG1GJyYrpX)**

1. Abra o link (é preciso uma conta Google). Em **Ambiente de execução > Executar tudo**, ou rode as células com `Shift+Enter`.
2. O notebook obtém o código, compila com `make`, roda os testes e executa os exemplos.

Se preferir montar o notebook do zero, crie um notebook novo e rode estas células:

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
!./mlc --ast tests/valid/fatorial.ml
!./mlc tests/invalid/spec_5_4.ml
```

Saída esperada: `make test` termina com `passaram: 17, falharam: 0`, e os três exemplos imprimem
`120`, `7` e `10`. O último comando mostra os erros sintáticos do exemplo 5.4 da especificação.

> Para o link funcionar para quem avaliar, compartilhe o notebook com **"Qualquer pessoa com o link"**
> (botão *Compartilhar* no canto superior direito do Colab).

## Executar na sua máquina

Requisitos: `g++` com suporte a C++17 (GCC 8 ou mais novo) e `make`.

```
make                       # gera o executável ./mlc
make test                  # roda os 17 testes
./mlc prog.ml              # só valida (léxico + sintático)
./mlc --tokens prog.ml     # lista os tokens com linha:coluna
./mlc --symbols prog.ml    # tabela de símbolos
./mlc --ast prog.ml        # árvore sintática abstrata
echo 5 | ./mlc --run tests/valid/fatorial.ml   # executa (entrada pela stdin)
./mlc --grammar            # produções, FIRST, FOLLOW e tabela LL(1)
```

Sem `make` (por exemplo, no Windows com MinGW/MSYS2):

```
g++ -std=c++17 -O2 -o mlc src/main.cpp src/lexer.cpp src/grammar.cpp src/parser.cpp src/ast.cpp src/interp.cpp
```

No terminal do Windows, se os acentos aparecerem quebrados, rode `chcp 65001` antes.

Códigos de saída: `0` ok · `1` erro léxico/sintático · `2` erro de execução · `3` uso incorreto.

## Estrutura

| Pasta/arquivo | Conteúdo |
|---|---|
| `src/lexer.*` | scanner: tokens, comentários `//`, espaços, erros léxicos com posição |
| `src/grammar.*` | produções LL(1), cálculo de FIRST/FOLLOW e da tabela (detecta conflitos) |
| `src/parser.*` | parser com pilha, árvore de derivação e recuperação de erros (modo pânico) |
| `src/ast.*` | AST e sua construção a partir da árvore de derivação |
| `src/interp.*` | interpretador semântico |
| `docs/grammar.md` | FIRST, FOLLOW e tabela LL(1), **gerados** pelo código (`./mlc --grammar`) |
| `jflap/*.jff` | AFDs dos tokens (id, number, keywords, assign_eq, relops, symbols, comment) |
| `tests/` | `valid/` (com `.in`/`.out`), `invalid/` e `runtime/` (com `.expect`) |
| `tools/gen_jflap.py` | gerador dos autômatos do JFLAP |

Os arquivos `.jff` abrem no JFLAP 7.1 (*File > Open*) e podem ser testados em *Input > Multiple Run*.
O autômato de espaços em branco não existe e o de comentário não cobre espaço nem `[`, porque o JFLAP
lê espaço/tab como transição λ e dá erro interno com `[`. O scanner em C++ trata esses casos normalmente.

## Decisões de projeto (pontos em que a especificação é inconsistente)

1. **Corpo de `then`/`else`/`do` sem `begin/end`.** A regra 6.1.4 exige blocos, mas o exemplo 5.2
   (`maximo`) usa `then write(a) else write(b)`. O parser aceita as duas formas
   (`<corpo> → <bloco> | <comando_ne>`).
2. **Else pendente.** A extensão acima cria o conflito clássico em `<condicional'>` × `else`, resolvido
   casando o `else` com o `if` mais próximo (documentado em `docs/grammar.md`).
3. **`_` em identificadores.** O exemplo 5.3 usa `soma_ate_zero`, então ID = `[a-z][a-z0-9_]*`.
4. **Associatividade.** A gramática LL(1) é recursiva à direita e faria `10-3-2` valer `10-(3-2)`.
   A AST é refeita com associatividade à esquerda: `10-3-2 = 5` e `8/2*2 = 8`
   (ver `tests/valid/associatividade.ml`).
5. **Comando vazio e `;`.** `<linhas> → <comando> (';' <comando>)*` com comando vazio permitido:
   cobre "o último comando do bloco não leva `;`" e `begin end`.
6. **Variáveis.** Tabela indexada por nome (e não só `a-z`), pois os exemplos usam `fat`, `soma`, `valor`.
   Variáveis não inicializadas valem 0.
7. **Faixa de valores.** 0 a 2³¹−1. Resultado negativo, estouro, divisão por zero, `read` sem entrada e
   laço sem fim (50 milhões de passos) são **erros de execução** com linha:coluna.
8. `int`, `{`, `}` e `,` são tokens reconhecidos, mas não fazem parte da gramática: aparecem como erro sintático.

## Recuperação de erros

Após um erro, o parser descarta tokens até `;`, `end`, `.` ou fim de arquivo, desempilha até o topo
conseguir tratar o token atual e continua (até 30 erros). Erro léxico ou sintático impede a geração da AST.
