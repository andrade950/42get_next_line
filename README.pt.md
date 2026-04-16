# 📖 get_next_line

> *Leitura de linhas a partir de um descritor de ficheiro, uma de cada vez.*

Um projecto da 42 School que implementa uma função capaz de ler a partir de um descritor de ficheiro e devolver uma linha por chamada — incluindo o carácter de nova linha — gerindo eficientemente o excedente de leitura através de variáveis estáticas.

---

## 📚 Índice

- [O que é o get\_next\_line?](#o-que-é-o-get_next_line)
- [Como Funciona](#como-funciona)
- [Estrutura do Projecto](#estrutura-do-projecto)
- [Descrição das Funções](#descrição-das-funções)
- [Gestão de Memória](#gestão-de-memória)
- [Parte Bónus — Múltiplos Descritores de Ficheiro](#parte-bónus--múltiplos-descritores-de-ficheiro)
- [Compilação](#compilação)
- [Exemplo de Utilização](#exemplo-de-utilização)
- [Casos Extremos Tratados](#casos-extremos-tratados)

---

## O que é o get_next_line?

O `get_next_line` é uma função em C com o seguinte protótipo:

```c
char *get_next_line(int fd);
```

Quando chamada repetidamente (por exemplo, dentro de um ciclo), lê o ficheiro apontado pelo descritor `fd` uma linha de cada vez. Cada chamada devolve:

- A próxima linha lida, **incluindo o carácter `\n`** (se existir).
- `NULL` quando não há mais nada a ler ou em caso de erro.

A função funciona tanto em ficheiros regulares como na entrada padrão (`stdin`).

---

## Como Funciona

O principal desafio do `get_next_line` é que uma única chamada a `read()` pode ler mais bytes do que uma linha contém. O excedente — tudo o que vem depois do `\n` — tem de ser guardado para a chamada seguinte. Isto é resolvido com uma **variável estática** — `left_str` — que persiste entre chamadas.

Eis o fluxo geral em cada chamada:

```
get_next_line(fd)
      │
      ▼
ft_read_to_left_str()          ← Lê de fd para left_str até encontrar '\n' ou EOF
      │
      │  left_str contém agora pelo menos uma linha completa (ou tudo até ao EOF)
      ▼
ft_get_line()                  ← Extrai exactamente uma linha de left_str
      │
      ▼
ft_new_left_str()              ← Remove a linha extraída de left_str (guarda o resto)
      │
      ▼
Devolve a linha ao chamador
```

### Passo a passo

1. **`ft_read_to_left_str(fd, left_str)`**  
   Aloca um buffer de `BUFFER_SIZE + 1` bytes e chama `read()` em ciclo, concatenando cada pedaço em `left_str` via `ft_strjoin`, até encontrar um `\n` em `left_str` ou até `read()` devolver `0` (EOF).

2. **`ft_get_line(left_str)`**  
   Percorre `left_str` até ao primeiro `\n` (inclusive) e aloca uma nova string contendo exactamente essa linha.

3. **`ft_new_left_str(left_str)`**  
   Remove a linha devolvida de `left_str`, liberta a string antiga e devolve apenas o restante (tudo após o `\n`), pronto para a próxima chamada.

---

## Estrutura do Projecto

```
get_next_line/
├── get_next_line.c            # Lógica principal — parte obrigatória
├── get_next_line.h            # Cabeçalho — protótipos e definição de BUFFER_SIZE
├── get_next_line_utils.c      # Funções auxiliares — parte obrigatória
├── get_next_line_bonus.c      # Lógica principal — bónus (múltiplos fd)
├── get_next_line_bonus.h      # Cabeçalho — parte bónus
└── get_next_line_utils_bonus.c# Funções auxiliares — parte bónus
```

---

## Descrição das Funções

### `get_next_line.c`

#### `get_next_line(int fd)`

```c
char *get_next_line(int fd);
```

Ponto de entrada. Valida o descritor de ficheiro e o `BUFFER_SIZE`. Chama `ft_read_to_left_str` para preencher o buffer estático, extrai uma linha com `ft_get_line` e actualiza o buffer restante com `ft_new_left_str`.

A chave é o `static char *left_str` — mantém o seu valor entre chamadas, funcionando como a "memória" do que já foi lido mas ainda não devolvido.

#### `ft_read_to_left_str(int fd, char *left_str)`

```c
char *ft_read_to_left_str(int fd, char *left_str);
```

Lê de `fd` em blocos de `BUFFER_SIZE` bytes, concatenando cada bloco em `left_str` com `ft_strjoin`. Para assim que um `\n` é encontrado em `left_str`, ou quando `read()` sinaliza EOF (`rd_bytes == 0`). Devolve `NULL` em caso de erro de leitura.

---

### `get_next_line_utils.c`

#### `ft_strlen(char *s)`
Devolve o comprimento de `s`. Devolve `0` se `s` for `NULL`.

#### `ft_strchr(char *s, int c)`
Percorre `s` à procura do carácter `c`. Devolve um apontador para a primeira ocorrência, ou `NULL`. Se `c == '\0'`, devolve um apontador para o terminador nulo. É usada para verificar se `left_str` já contém uma nova linha.

#### `ft_strjoin(char *left_str, char *buff)`
Concatena `left_str` e `buff` numa nova string alocada dinamicamente. Liberta `left_str` antes de devolver, pois é feita uma nova alocação. Se `left_str` for `NULL` (primeira chamada), é inicializada como uma string vazia `""`.

#### `ft_get_line(char *left_str)`
Aloca e devolve uma string com uma linha de `left_str` — tudo até ao primeiro `\n` inclusive, ou até `\0` se não existir nova linha. Devolve `NULL` se `left_str` estiver vazia.

#### `ft_new_left_str(char *left_str)`
Calcula quantos caracteres foram consumidos pela linha devolvida, avança para além do `\n` e devolve uma nova string com apenas o restante. Liberta o `left_str` antigo. Devolve `NULL` se o buffer estiver esgotado (não restam caracteres após a linha).

---

## Gestão de Memória

Todas as alocações são controladas e libertadas correctamente:

| Alocação | Libertada por |
|---|---|
| `buff` em `ft_read_to_left_str` | Sempre libertado antes de retornar |
| `left_str` dentro de `ft_strjoin` | Libertado e substituído pela nova string concatenada |
| `left_str` no EOF | Libertado dentro de `ft_new_left_str` |
| `line` devolvida | **Responsabilidade do chamador** — deve chamar `free()` |

> ⚠️ O chamador é sempre responsável por libertar a string devolvida por `get_next_line`.

---

## Parte Bónus — Múltiplos Descritores de Ficheiro

A versão bónus utiliza um **array estático** de apontadores em vez de um único apontador estático:

```c
static char *left_str[4096];
```

Cada descritor de ficheiro é usado como índice nesse array. Desta forma, a função gere independentemente a posição de leitura de até 4096 descritores em simultâneo, sem interferências entre eles.

```c
left_str[fd] = ft_read_to_left_str(fd, left_str[fd]);
line = ft_get_line(left_str[fd]);
left_str[fd] = ft_new_left_str(left_str[fd]);
```

Esta abordagem satisfaz os dois requisitos do bónus:
- Utiliza apenas **uma variável estática** (um array conta como uma variável).
- Gere correctamente **múltiplos descritores de ficheiro** alternando entre chamadas.

---

## Compilação

Compile com um tamanho de buffer personalizado usando a flag `-D BUFFER_SIZE=n`:

```bash
# Parte obrigatória
cc -Wall -Wextra -Werror -D BUFFER_SIZE=42 get_next_line.c get_next_line_utils.c -o gnl

# Parte bónus
cc -Wall -Wextra -Werror -D BUFFER_SIZE=42 get_next_line_bonus.c get_next_line_utils_bonus.c -o gnl_bonus
```

O `BUFFER_SIZE` por omissão (quando a flag não é fornecida) é `1000`, conforme definido no cabeçalho.

---

## Exemplo de Utilização

```c
#include <fcntl.h>
#include <stdio.h>
#include "get_next_line.h"

int main(void)
{
    int     fd;
    char    *line;

    fd = open("ficheiro.txt", O_RDONLY);
    if (fd < 0)
        return (1);
    while ((line = get_next_line(fd)) != NULL)
    {
        printf("%s", line);
        free(line);
    }
    close(fd);
    return (0);
}
```

---

## Casos Extremos Tratados

| Caso | Comportamento |
|---|---|
| `fd < 0` | Devolve `NULL` imediatamente |
| `BUFFER_SIZE <= 0` | Devolve `NULL` imediatamente |
| Erro em `read()` | Liberta o buffer, devolve `NULL` |
| Ficheiro vazio | `ft_get_line` devolve `NULL` com `left_str` vazio |
| Ficheiro sem `\n` final | Última linha devolvida sem `\n`, chamada seguinte devolve `NULL` |
| `BUFFER_SIZE` muito grande | Funciona — lê tudo em um ou poucos blocos |
| `BUFFER_SIZE = 1` | Funciona — lê um byte de cada vez |
| `left_str` a `NULL` na primeira chamada | `ft_strjoin` inicializa-o como `""` |
| Múltiplos fd (bónus) | Cada fd tem a sua posição independente no array estático |

---