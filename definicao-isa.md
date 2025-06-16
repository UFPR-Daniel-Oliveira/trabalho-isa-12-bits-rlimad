# Definição da ISA de 12 bits

**Autor:** Rafael Lima Dias  
**GRR:** 20244378 
**Orientador:** Daniel Oliveira   
**Instituição:** Universidade Federal do Paraná (UFPR)

---

# Sumário

1. [Introdução](#1.Introdução)  
2. [Requisitos e Restrições](#requisitos-e-restricoes)  
3. [Decisões de Projeto e Trade-offs](#decisoes-de-projeto-e-trade-offs)  
4. [Conjunto de Registradores](#conjunto-de-registradores)  
5. [Formatos de Instrução](#formatos-de-instrucao)  
6. [Tabela de Instruções (Green Card)](#tabela-de-instrucoes-green-card)  
7. [Exemplos de Codificação](#exemplos-de-codificacao)  
8. [Implementações](#implementacoes)
   
---

## 1. Introdução

### 1.1 Contexto

No curso de Projetos Digitais e Microprocessadores, fomos instruídos a realizar como trabalho de conclusão da disciplina um microprocessador de acordo com os ensinamentos advindos da classe. Logo, para cumprir com as expectativas da tarefa se torna necessário a definição de uma ISA. A **ISA** (Instruction Set Architecture) define o “contrato” entre hardware e software, especificando como cada instrução é codificada em bits, que registradores existem, como os dados são movimentados e quais operações são suportadas.  

Este trabalho propõe uma ISA simplificada de **12 bits** inspirada em ideias de RISC-V e do clássico PDP-8, objetivando projetar e implementar uma microarquitetura monociclo no Logisim-evolution.Tendo essa limitação de 12 bits é preciso estar atento para equilibrar o número de registradores, campos de opcode, immediatos e offsets, partindo do pressuposto de um espaço muito restrito.

### 1.2 Objetivos do documento

Este relatório tem por objetivo:

1. **Definir** completamente a ISA de 12 bits, incluindo:  
   - Conjunto de registradores e convenções de uso.  
   - Formatos de instrução (R-type, I-type, J-type, Jext).  
   - Codificação de cada instrução (opcode, funct, campos de registrador, immediato/offset).  
2. **Justificar** todas as decisões de projeto e trade-offs, especialmente:  
   - Escolha de 4 registradores de 12 bits (R0…R3).  
   - Alocação de 3 bits para opcode e 2 bits por registrador ou 3 bits de imediato.  
3. **Fornecer** exemplos de codificação e pseudo-código de alto nível para ilustrar a aplicação da ISA em programas com IF/ELSE, loops e chamadas de função.  

### 1.3 Visão geral da ISA de 12 bits

A ISA proposta apresenta:

- **4 registradores** de 12 bits cada (R0…R3), onde R0 é fixo em zero, R1 e R2 são temporários (caller-saved) e R3 serve como ponteiro de pilha ou registrador de argumento/retorno (callee-saved).  
- Instruções de **12 bits** fixos, sem formatos variáveis. Cada instrução é decodificada em 3 bits de opcode e campos restantes que variam conforme o formato.  
- Quatro **formatos** principais:  
  1. **R-type:** operações entre registradores (ADD, SUB, AND, OR, XOR, SHL, SHR, NOP).  
  2. **I-type:** registrador + imediato (ADDI, SUBI) ou acesso a memória (LD, STR) com imediato de 3 bits (–4..+3).  
  3. **J-type curto:** desvio condicional (BEQ, BNE) com offset de 3 bits.    
- Processo de **sign-extend** para estender immediatos e offsets a 12 bits.  

---

## 2. Requisitos e Restrições

### 2.1 Tamanho fixo de instrução (12 bits)  
Cada instrução desta ISA ocupa exatamente **12 bits** em memória de instruções. Não há formatos de largura variável: todas as instruções — aritméticas, lógicas, de memória, desvio ou salto — devem ser codificadas em 12 bits. Esse limite impõe um trade-off rígido entre número de bits dedicados a opcode, registradores e campos de imediato/offset.

### 2.2 O que é um Trade-off?

Um **trade-off** é um **compromisso** entre duas ou mais características opostas. Aqui, por exemplo:

- Reservar **mais bits para registradores** (ex.: usar 8 registradores, 3 bits cada) reduz o espaço para **imediatos** ou **offsets**, limitando o alcance de constantes e saltos.  
- Reservar **mais bits para opcode** aumenta o número de instruções distintas, mas reduz bits disponíveis para campos de registrador e imediato.

Em 12 bits totais, precisamos **equilibrar**:
1. **Número de registradores** (2 vs. 3 bits por índice).  
2. **Bits de imediato/offset** (alcance de constantes e saltos).  
3. **Bits de opcode** (quantos tipos de instruções queremos).

### 2.3 O que é um Offset?

Um **offset** é um **deslocamento relativo** usado para saltos ou cálculo de endereço. Em nossa ISA:

- **Offset de desvio (off3)**  
  - Tem **3 bits** com sinal (faixa de –4 a +3).  
  - É adicionado ao **PC** (program counter) para saltar a instrução desejada.

- **Offset de memória (imm3)**  
  - Também 3 bits com sinal, usado em `LD` e `STR` como deslocamento em relação ao conteúdo de um registrador-base.

Em ambos os casos, antes de usar, fazemos **sign-extend** (estender o bit de sinal) para 12 bits, garantindo que o valor negativo ou positivo seja interpretado corretamente.  

Esse equilíbrio — ou trade-off — guia todas as escolhas de formatação de instrução.

### 2.4 Arquitetura *load/store*  
A CPU segue o modelo *load/store*:  
- Todas as operações aritméticas e lógicas ocorrem **exclusivamente** entre valores residentes em registradores.  
- A memória de dados só é acessada por instruções de load (`LD`) ou store (`STR`), que movem valores **entre** registradores e memória.  
- Não existem instruções que façam, em um único passo, cálculo e acesso à memória simultâneo.

### 2.5 Word-size, registradores e ULA de 12 bits  
- **Tamanho de palavra**: 12 bits.  
- **Registradores**: 4 registradores de 12 bits cada (R0…R3).  
- **ULA**: opera em palavras de 12 bits, realizando soma, subtração, lógicas e shifts em 12 bits, com resultado truncado a 12 bits.  
- Todos os dados internos e os caminhos do datapath (barramentos, registradores, ALU) têm largura de 12 bits.

### 2.6 Endereçamento de memória  
m nossa ISA, cada **palavra** (word) tem **12 bits** e, conceitualmente, tanto a **memória de instruções** quanto a **memória de dados** são _word-addressable_ (um índice → um word de 12 bits).  
Porém, no **Logisim-evolution** não existe memória nativa de 12 bits: você só encontra memórias de **8 bits** (byte-addressable) ou, opcionalmente, memórias configuráveis para **16 bits**. Para implementar nosso modelo:

1. **Usando memória de 16 bits**  
   - Configure a RAM/ROM para 16 bits de largura por célula.  
   - Cada endereço retorna 16 bits; **usamos apenas 12** dos bits (por exemplo, os 12 mais significativos) e ignoramos ou deixamos fixos os 4 bits excedentes.  
   - **Não há conversão de endereço**: se o registrador contém o índice de word (0, 1, 2…), basta usá-lo diretamente como endereço.

2. **Usando memória de 8 bits (byte-addressable)**  
   - Cada célula armazena apenas 8 bits.  
   - Para guardar uma palavra de 12 bits, precisamos **2 bytes** (2 × 8 bits = 16 bits), sobrando 4 bits.  
   - **Mapeamento de endereço**:  
     - O registrador calcula um **índice de word** (0, 1, 2…).  
     - Para converter em **índice de byte**, multiplicamos por 2:
       
       ```text
       byte_address = word_address * 2
       ```  
     - Exemplo: se `word_address = 5`, então  
       - primeiro byte em `byte_address = 5 * 2 = 10`  
       - segundo byte em `byte_address + 1 = 11`

#### Por que multiplicar por 2?

- Porque estamos embalando cada word de 12 bits em **2 bytes** sequenciais.  
- A memória de 8 bits usa índices de byte, e cada palavra ocupa dois desses índices.  
- Assim, todo acesso a word (load/store) segue:
  
  ```c
  word_index   = R[rs] + signExtend(offset3);
  byte_address = word_index * 2;    // converte word → byte
  // então lê/escreve nos dois bytes consecutivos e remonta o valor de 12 bits
  ```
  
#### Por que 16 bits?

- Porque 16 bits (2 bytes) é o mínimo que acomoda nossos 12 bits de forma contígua.  
- O Logisim não tem memórias de 12 bits “prontas”, então usamos 16 bits e reservamos 4 bits extras (ignorados ou sempre zero).

Dessa forma, mantemos a visão conceitual de memória word-addressable de 12 bits, enquanto respeitamos as restrições físicas do Logisim.

### 2.7 Conjunto mínimo de instruções  
Para satisfazer os requisitos do enunciado, a ISA deve incluir:  
- **Instruções aritméticas**: ADD, SUB.  
- **Instruções lógicas**: AND, OR, XOR.  
- **Instruções de shift**: SHL (left), SHR (right).  
- **Instruções de desvio condicional**: BEQ, BNE.  
- **Instruções de salto**: uso de BEQ R0,R0 para salto curto.
- **Instruções de memória**: LD (load de 12 bits), STR (store de 12 bits).  

### 2.8 Suporte a estruturas de programação  
A ISA deve permitir a implementação em nível de assembly de:  
- **IF / ELSE**: comparações seguidas de BEQ/BNE e saltos curtos.  
- **Laços** (*FOR*, *WHILE*): usando decremento/incremento de registrador e BEQ/BNE para controle de repetição.  
- **Chamadas de função**: passagem de parâmetros em registradores, uso de R3 como *link* ou ponteiro de pilha, e retorno via salto incondicional (BEQ R0,R0) ou Jext.  

## 3. Decisões de Projeto e Trade-offs

### 3.1 Quantidade de registradores: 4 vs 8  
- **8 registradores (3 bits)**  
  - Permite até 8 variáveis em registradores, reduz acessos à memória.  
  - Mas, para cada campo de registrador numa instrução, você gasta 3 bits. Em R-type (que precisa de 3 campos: rd, rs, rt), já são 9 bits só em registradores — sobra apenas 3 bits para opcode/imediato!  
- **4 registradores (2 bits)**  
  - Cada campo de registrador ocupa apenas 2 bits; em R-type são 6 bits no total, deixando 6 bits para opcode/funct ou imediato.  
  - Trade-off: menos registradores → mais load/store e mais “spill” na memória em códigos complexos.

### Por que usamos 2 bits para cada registrador?

Optamos por **4 registradores** (R0..R3) para caber em 12 bits:

- Cada índice de registrador ocupa **2 bits** (00, 01, 10, 11).  
- Se tivéssemos 8 registradores, seriam **3 bits** por índice — problemas de espaço em R-type (3 campos × 3 bits = 9 bits só para registradores!).

Com 2 bits por registrador, num R-type:
- **rd (2 bits)** + **rs (2 bits)** + **rt (2 bits)** = 6 bits  
- + **opcode (3)** + **funct3 (3)** = 12 bits exatamente.

**Escolha final**: 4 registradores, para equilibrar flexibilidade de instrução e alcance de imediato/offset em 12 bits.

### 3.2 Convenção de uso dos registradores (R0…R3)  
| Registrador | Índice | Uso principal                 | Saved      |
|:-----------:|:------:|:------------------------------|:----------:|
| **R0**      | `00`   | T0 (temporário)      | —          |
| **R1**      | `01`   | T1 (temporário)                | Caller-saved |
| **R2**      | `10`   | T2 (temporário)                | Caller-saved |
| **R3**      | `11`   | SP / A0 (pilha)                | Caller-saved |

Quando escrevemos **sub-rotinas** (funções), precisamos decidir quem preserva o conteúdo dos registradores:

- **Caller-saved** (salvo pelo chamador):
  - Registradores que **quem chama** a função deve salvar (por exemplo, na pilha) se quiser manter seus valores após a chamada.
  - **R1 e R2** são temporários, usados a gosto pela função chamada; o chamador deve restaurá-los depois.

- **Callee-saved** (salvo pela função chamada):
  - Registradores que a **função chamada** deve preservar. Se a função for usar, ela mesma salva e restaura antes de retornar.
  - **R3** (SP / A0) é callee-saved, garantindo que o chamador possa confiar nele sem precisar salvar antes.

### 3.3 Distribuição de bits em 12 bits  
Para caber R-type, I-type e J-type em 12 bits, reservamos:

- **3 bits** para **opcode** (até 8 grupos de instruções)  
- **2 bits** para cada registrador ou parte de imediato/offset  
- **3 bits** para **funct3** (em R-type) ou para **imediato/offset** (em I/J-type)  

#### Exemplo de particionamento

Em todas as instruções, contamos de **bit 11** (mais significativo) até **bit 0** (menos significativo). 

| Formato   | Bits [11..9]     | Bits [8..6]             | Bits [5..4]                       | Bits [3..2]   | Bits [1..0]       |
|:---------:|:----------------:|:-----------------------:|:---------------------------------:|:-------------:|:-----------------:|
| **R-type**| `opcode` (3 bits)| `funct3` (3 bits)       | `rd` (2 bits)                     | `rs` (2 bits) | `rt` (2 bits)     |
| **I-type**| `opcode` (3 bits)| `imm3` (3 bits)         | `rd` (2 bits)                     | `rs` (2 bits) | `rt` (2 bits)     |
| **J-type**| `opcode` (3 bits)| `off3` (3 bits)         | `codificação` (1 bit) + `0` (1 bit)| `rs` (2 bits) | `rt` (2 bits)     |

- **opcode** (bits 11–9): identifica o formato/instrução.  
- **funct3** (bits 8–6, em R-type): seleciona, dentro do grupo R, qual operação (ADD, SUB, AND...).  
- **rd, rs, rt** (2 bits cada): índices dos 4 registradores disponíveis (R0..R3).
  - **rd** = Registrador Desino.
  - **rs** = Primeiro Registrador Fonte
  - **rt** = Segundo Registrador Fonte 
- **imm3, off3** (3 bits): imediato ou deslocamento de –4..+3 (signed).  
- **pad** são bits reservados **sem função ativa** (sempre zero).  
  - Em I-type, **pad = bits [1..0]** sempre `00`.  
  - Em J-type, **pad = bit [0]** sempre `0`.  

  Servem para:
  1. **Alinhamento**: manter os campos principais em posições fixas.  
  2. **Futuras extensões**: permitir adicionar “funct” extras ou mais immediatos sem mudar todo o esquema.

### 3.4 Justificativa para os formatos escolhidos  
1. **R-type (3 bits opcode + 3 bits funct3 + 3×2 bits registradores)**  
   - Garante até 8 operações entre registradores (ADD, SUB, AND, OR, XOR, SHL, SHR, NOP).  
2. **I-type (3 bits opcode + 2×2 bits registradores + 3 bits immed.)**  
   - Suporta ADDI, SUBI e load/store com imediato de ±4. O padding de 2 bits permite futuras extensões.  
3. **J-type curto (opcode=`110`)**  
   - BEQ/BNE em um único formato, usando 1 bit para condição e 3 bits de offset (±4 instruções).  
4. **Jext opcional (opcode=`111` + 9 bits offset)**  
   - Para saltos mais longos (±256 instruções) sem encadear vários BEQ.  

## 4. Conjunto de Registradores

### 4.1 Tabela de Registradores

| Registrador | Índice (bin) | Função / Descrição                                 | Salvo           |
|:-----------:|:------------:|:---------------------------------------------------|:---------------:|
| **R0**      | `00`         | Zero constante (sempre 0)                           | —               |
| **R1**      | `01`         | T0 / 1º argumento / valor de retorno                | Caller-saved    |
| **R2**      | `10`         | T1 / 2º argumento                                   | Caller-saved    |
| **R3**      | `11`         | Stack Pointer (SP) em _words_ (cada word = 12 bits) | Callee-saved    |

### 4.2 R0: Registrador Zero

- **Sempre vale 0**; escritas em R0 são ignoradas.  
- Facilita carregar constantes pequenas e saltos incondicionais:

  ```asm
  ADDI R1, R0, 3    # R1 ← 0 + 3
  BEQ  R0, R0, +4   # salto curto incondicional
  ```
### 4.3 R1 e R2: Temporários / Argumentos  

- **R1 (T0**) e **R2 (T1)** servem tanto a cálculos intermediários quanto a primeiro e segundo argumentos de função.
- **Caller-saved**: quem chama deve salvar/restaurar R1/R2 se precisar deles após a chamada.

### 4.4 R3: Stack Pointer (SP)

- **R3** aponta para o topo da pilha em unidades de word (12 bits).
- **Callee-saved**: toda função que modificar R3 deve salvar e restaurar seu valor.

### 4.5 Convenções de Chamada de Função

1. **Passagem de parâmetros**  
   - Até 2 parâmetros em `R1` e `R2`.  
2. **Valor de retorno**  
   - A função coloca o resultado em **R1**.
   - Após o retorno, o chamador lê `R1`. 
3. **Fluxo de chamada/retorno**  

   ```asm
     # --- Chamador ---
    ADDI R1, R0, 10       # arg1 = 10
    ADDI R2, R0, 20       # arg2 = 20
    BEQ  R0, R0, func     # call func
    # após retornar, R1 contém o resultado
    
    # --- Função ---
    func:
      ADD  R1, R1, R2     # R1 ← arg1 + arg2
      BEQ  R0, R0, ret    # return
    
    ret:
   ```

### 4.6 Endereçamento de Pilha e Uso de Offsets

- **SP em words**, não em bytes. Cada push/pop move SP em 1 word: 

```asm
# push Rn
ADDI R3, R3, -1      # SP ← SP - 1
STR  Rn, [R3 + 0]    # MEM[word = SP] ← Rn

# pop Rn
LD   Rn, [R3 + 0]    # Rn ← MEM[word = SP]
ADDI R3, R3, +1      # SP ← SP + 1
```

- No **Logisim** , a memória é byte-addressable (8 bits por célula). Cada word de 12 bits ocupa 2 bytes (16 bits físicos), sobrando 4 bits não usados. Internamente, o datapath faz:

```ini
byte_address = word_address * 2
```

logo, só precisa usar `STR/LD [R3+imm3]` em assembly.

## 5. Formatos de Instrução

Cada instrução ocupa **12 bits** fixos, numerados de **11** (bit mais significativo) a **0** (bit menos significativo). Dependendo do opcode, os bits são interpretados em quatro formatos principais.

### 5.1 Formato R-type

Operações registrador→registrador (ADD, SUB, AND, OR, XOR, SHL, SHR, NOP):

| Formato   | Bits [11..9]     | Bits [8..6]             | Bits [5..4]   | Bits [3..2]   | Bits [1..0]       |
|:---------:|:----------------:|:-----------------------:|:-------------:|:-------------:|:-----------------:|
| **R-type**| opcode (3 bits)  | `funct3` (3 bits)       | `rd` (2 bits) | `rs` (2 bits) | `rt` (2 bits)     |

| Campo  | Bits  | Descrição                                         |
| :----- | :---- | :------------------------------------------------ |
| opcode | 11..9 | `000` → R-type                                    |
| funct3 | 8..6  | Seleciona operação (ADD=000, SUB=001, …, NOP=111) |
| rd     | 5..4  | Registrador destino (R0..R3)                      |
| rs     | 3..2  | Registrador fonte 1 (R0..R3)                      |
| rt     | 1..0  | Registrador fonte 2 (R0..R3)                      |

**Exemplo:**
```asm
ADD R1, R2, R3   # 000 000 01 10 11
```

### 5.2 Formato I-type

Imediato de 3 bits ou load/store (ADDI, SUBI, LD, STR):

Imediato de 3 bits ou load/store (ADDI, SUBI, LD, STR):

| Formato   | Bits [11..9]       | Bits [8..6]      | Bits [5..4]                                    | Bits [3..2]                                           | Bits [1..0]                                             |
|:---------:|:------------------:|:----------------:|:----------------------------------------------:|:-----------------------------------------------------:|:------------------------------------------------------:|
| **I-type**| `opcode` (3 bits)  | `imm3` (3 bits)  | `rd` (2 bits)                                  | `rs` (2 bits)                                         | `rt` (2 bits)                                           |


| Campo  | Bits  | Descrição                                                                   |
| :----- | :---- | :--------------------------------------------------------------------------- |
| opcode | 11..9 | `011` = ADDI, `100` = SUBI, `101` = LD, `110` = STR                          |
| imm3   | 8..6  | Imediato com sinal (–4..+3)                                                 |
| rd     | 5..4  | Registrador destino (para ADDI, SUBI e LD); **não usado** em STR            |
| rs     | 3..2  | Registrador fonte (para ADDI/SUBI) / Registrador base (para LD/STR)         |
| rt     | 1..0  | Registrador fonte (para STR); **não usado** em ADDI, SUBI e LD (reservado)  |

**Exemplo:**
```asm
ADDI R2, R1, -2  # 011 10 01 110 00
```

### 5.3 Formato J-type Curto

Branch condicional BEQ/BNE com offset de 3 bits:

| Formato   | Bits [11..9]       | Bits [8..6]      | Bits [5..4]                           | Bits [3..2]   | Bits [1..0]       |
|:---------:|:------------------:|:----------------:|:-------------------------------------:|:-------------:|:-----------------:|
| **J-type**| `opcode` (3 bits)  | `off3` (3 bits)  | `codificação` (1 bit) + `0` (1 bit)   | `rs` (2 bits) | `rt` (2 bits)     |

**Exemplo:**
```asm
BNE R0, R1, -1  # 110 1 00 01 111 0
```

### 5.5 Sign-extrend de Imediato/Offset

- **imm3, off3** (3 bits → 12 bits): repetir o bit de sinal (b₂) em [11..3].

**Exemplo:**
```text
imm3 = 110₂ (–2)  
sign-extend12 = 111111111110₂ (–2)
```

## 6. Tabela de Instruções (Green Card)

### 6.1 Instruções R-type (opcode = `000`)

| funct3 | Instrução       | Codificação (12 bits)       | Semântica                                 |
|:------:|:----------------|:----------------------------|:------------------------------------------|
| `000`  | `ADD rd, rs, rt`| `000 000 rd rs rt`          | `R[rd] = R[rs] + R[rt]`                   |
| `001`  | `SUB rd, rs, rt`| `000 001 rd rs rt`          | `R[rd] = R[rs] - R[rt]`                   |
| `010`  | `AND rd, rs, rt`| `000 010 rd rs rt`          | `R[rd] = R[rs] & R[rt]`                   |
| `011`  | `OR rd, rs, rt` | `000 011 rd rs rt`          | ``R[rd] = R[rs] \| R[rt]``                |
| `100`  | `XOR rd, rs, rt`| `000 100 rd rs rt`          | `R[rd] = R[rs] ^ R[rt]`                   |
| `101`  | `SHL rd, rs, rt`| `000 101 rd rs rt`          | `R[rd] = R[rs] << (R[rt] & 0xF)`          |
| `110`  | `SHR rd, rs, rt`| `000 110 rd rs rt`          | `R[rd] = R[rs] >> (R[rt] & 0xF)`          |
---

### 6.2 Instruções I-type (opcode = `001`, `010`, `011`, `100`)

| opcode | Instrução              | Codificação (12 bits) | Semântica                            |
| :----: | :--------------------- | :-------------------- | :----------------------------------- |
|  `001` | `ADDI rd, rs, imm3`    | `001 imm3 rd rs 00`   | `R[rd] = R[rs] + signext(imm3)`      |
|  `010` | `SUBI rd, rs, imm3`    | `010 imm3 rd rs 00`   | `R[rd] = R[rs] - signext(imm3)`      |
|  `011` | `LD   rd, [rs + imm3]` | `011 imm3 rd rs 00`   | `R[rd] = MEM[R[rs] + signext(imm3)]` |
|  `100` | `STR  rt, [rs + imm3]` | `100 imm3 00 rs rt`   | `MEM[R[rs] + signext(imm3)] = R[rt]` |

### 6.3 Instruções J-type curto (opcode = `101`)

| cond | Instrução          | Codificação (12 bits) | Semântica                                                     |
| :--: | :----------------- | :-------------------- | :------------------------------------------------------------ |
|  `0` | `BEQ rs, rt, off3` | `101 off3 0 0 rs rt`  | if `R[rs] == R[rt]` then `PC += signext(off3)` else `PC += 1` |
|  `1` | `BNE rs, rt, off3` | `101 off3 1 0 rs rt`  | if `R[rs] != R[rt]` then `PC += signext(off3)` else `PC += 1` |

> **Nota:** salto incondicional curto pode usar `BEQ R0, R0, off3`.


**Legenda:**  
- `opcode`, `funct3`, `cond`, `rd`, `rs`, `rt`, `imm3`, `off3` mostram apenas os campos; cada `rd`, `rs`, `rt` representa 2 bits para índices 00..11 (R0..R3).  
- `signext(...)` = estender o bit de sinal ao múltiplo de 12 bits.

## 7. Exemplos de Codificação

Para cada instrução, seguimos estes passos:

1. Identificar o **formato** (R-type, I-type, J-type curto ou Jext).  
2. Anotar os valores de cada campo (opcode, funct3/cond, rd/rs/rt, imm3/off3/offset9).  
3. Converter cada campo para binário com o número de bits correto.  
4. Concatenar na ordem dos bits [11..0] para obter o código de 12 bits.

### 7.1 `ADD R1, R2, R3` (R-type)

1. **Formato**: R-type → `opcode = 000` (bits 11..9)  
2. **funct3**: ADD → `000` (bits 8..6)  
3. **rd**: R1 → `01` (bits 5..4)  
4. **rs**: R2 → `10` (bits 3..2)  
5. **rt**: R3 → `11` (bits 1..0)  

| Campo   | Valor Decimal/Binário | Bits  |  
|:--------|:----------------------:|:------|  
| opcode  | 0 → `000`              | 11..9 |  
| funct3  | 0 → `000`              |  8..6 |  
| rd      | 1 → `01`               |  5..4 |  
| rs      | 2 → `10`               |  3..2 |  
| rt      | 3 → `11`               |  1..0 |  

**Concatenação**: 000 000 01 10 11 = 000000011011₂

### 7.2 `ADDI R2, R1, -2` (I-type)

1. **Formato**: Formato: I-type → opcode (3) │ imm3 (3) │ rd (2) │ rs (2) │ rt (2)
2. **opcode**: R2 → `10` (bits 8..7)  
3. **imm3**: –2 → em 3 bits signed:
   +2 = 010 → –2 = dois-complemento → 110 (bits 8..6)
4. **rd**: R2 → 10 (bits 5..4)  
5. **rs**: R1 → 01 (bits 3..2)
6. **rt**: reservado → 00 (bits 1..0)

|  Campo | Valor |  Bits |
| :----: | :---: | :---: |
| opcode | `011` | 11..9 |
|  imm3  | `110` |  8..6 |
|   rd   |  `10` |  5..4 |
|   rs   |  `01` |  3..2 |
|   rt   |  `00` |  1..0 |

**Concatenação**: 011 110 10 01 00 = 011110100100

### 7.3 `BNQ R0, R1, -1` (J-type curto)

1. **Formato**: J-type curto → opcode | off3 | codificação | pad=0 | rs | rt
2. **opcode**: BNQ → `101` (bits 11..9)
3. **off3**: –1 → em 3 bits signed: 111 (bits 8..6)
4. **condição**: cond = 1 (bit 5)
5. **pad**: sempre 0 (bit 4)
6. **rs**: R0 → 00 (bits 3..2)
7. **rt**: R1 → 01 (bits 1..0)

|      Campo      | Valor |  Bits |
| :-------------: | :---: | :---: |
|    **opcode**   | `101` | 11..9 |
|     **off3**    | `111` |  8..6 |
| **codificação** |  `1`  |   5   |
|     **pad**     |  `0`  |   4   |
|      **rs**     |  `00` |  3..2 |
|      **rt**     |  `01` |  1..0 |

**Concatenação**: 101 111 1 0 00 01 = 101111100001₂

## 8. Implementações

### 8.1 Implementando IF/ELSE

```asm
# if (R1 != 0) then … else …
    BNE R1, R0, THEN_LABEL    # se R1 != 0, vá para THEN_LABEL
    # bloco ELSE
    …                        # instruções do else
    BEQ R0, R0, END_IF       # pula o bloco then
THEN_LABEL:
    # bloco THEN
    …                        # instruções do then
END_IF:
    # continua execução
```
- `BNE R1, R0, THEN_LABEL` faz salto condicional curto se `R1 != 0`.
- `BEQ R0, R0, END_IF` é salto incondicional curto (sempre verdadeiro).

### 8.2 Construindo Loops (FOR / WHILE)

**Exemplo** `FOR: for (i = 0; i < 10; i++) …`

```asm
    # inicializa i = 0
    ADDI R2, R0, 0       # R2 = i

LOOP:
    # testa i < 10  → se i == 10, sai
    SUBI R3, R2, 10      # R3 = i - 10
    BEQ  R3, R0, END_LOOP  # se R3 == 0, i == 10 → END_LOOP

    # corpo do loop
    …                    # suas instruções aqui

    # incrementa i
    ADDI R2, R2, 1       # i++

    # salto de volta ao início do loop
    BEQ  R0, R0, LOOP    # salto incondicional curto

END_LOOP:
    # continua execução
```

### 8.3 Chamadas de Função

**Convenção:**

- Parâmetros em R1, R2.
- Retorno em R1.
- SP em R3 (caller/callee-saved conforme definido).

```asm
# --- Chamador ---
    ADDI R1, R0, X       # 1º argumento = X
    ADDI R2, R0, Y       # 2º argumento = Y
    BEQ  R0, R0, FUNC    # call FUNC (short jump)

    # após retorno, resultado em R1
    …                    # usa R1

# --- Função FUNC ---
FUNC:
    # corpo da função — por exemplo, soma:
    ADD   R1, R1, R2     # R1 = arg1 + arg2

    # retorno:
    BEQ   R0, R0, RET    # jump curto de volta

RET:
    # volta para o chamador
```

- A função escreve o resultado em R1 e faz salto curto de volta ao rótulo `RET`.
- O chamador sabe que, após o `BEQ R0,R0, FUNC`, a próxima instrução a executar (no `RET`) é logo após esse `BEQ`.
