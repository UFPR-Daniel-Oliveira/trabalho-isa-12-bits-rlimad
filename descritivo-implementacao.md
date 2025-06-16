# Descritivo da ISA e Microarquitetura

Este documento descreve as principais decisões de projeto da ISA de 12 bits e explica o funcionamento de cada unidade funcional da microarquitetura mostrada no diagrama.

---

## 1. Decisões de Projeto

1. **Largura de instrução fixa (12 bits)**  
   - Simplifica o fetch e a decodificação, mantendo todos os campos em posições fixas.

2. **Conjunto reduzido de registradores**  
   - 4 registradores de 12 bits cada (R0…R3).  
   - R0 sempre zero; R1/R2 caller-saved; R3 callee-saved/SP.

3. **Arquitetura Load/Store**  
   - Operações aritméticas e lógicas são somente entre registradores.  
   - Acesso à memória só via `LD`/`STR` com deslocamento de 3 bits (–4…+3).

4. **Três formatos principais de instrução**  
   - **R-type** (opcode + funct3 + rd + rs + rt)  
   - **I-type** (opcode + imm3 + rd + rs + rt)  
   - **J-type curto** (opcode + off3 + cond + pad + rs + rt)  

5. **Offset e imediato de 3 bits com sinal**  
   - Alcance de –4 a +3, estendido (sign-extend) a 12 bits.

6. **Branch relativo**  
   - BEQ/BNQ calculam `PC ← PC+1 + signext(off3)`, permitindo loops curtos sem instruções extras.

---

## 2. Unidades Funcionais

### 2.1 Program Counter (PC)  
- **Tipo:** Registrador síncrono de 12 bits  
- **Função:** Armazena endereço da próxima instrução.  
- **Entradas:**  
  - `clk`  
  - `nextPC` (12 bits)  
- **Saída:**  
  - `PC` → endereço para a ROM de instruções.

### 2.2 Memória de Instruções (ROM)  
- **Tipo:** ROM word-addressable, 12 bits por palavra  
- **Função:** Armazena o programa.  
- **Entradas:**  
  - `PC[11..0]`  
- **Saída:**  
  - `IR_in[11..0]` → entra num registrador de instrução.

### 2.3 Unidade de Controle (Control Unit)  
- **Tipo:** Lógica combinacional  
- **Função:** Gera sinais de controle a partir de `opcode`, `funct3` e `cond`:  

| opcode | opULA | WBReg | ALUSrc (MuxULA) | dado_breg | signal_control | PCSel |
|:------:|:-----:|:-----:|:---------------:|:--------:|:--------:|:-----:|
| R-type `000`  | `000` | 1     | 0               | 0        | 0        | 0     |
| addi `001`  | `001` | 1     | 1               | 0        | 0        | 0     |
|  subi `010`  | `010` | 1     | 1               | 0        | 0        | 0     |
| ld `011`  | `100` | 1     | 1               | 1        | 0        | 0     |
| store `100`  | `101` | 0     | 1               | 0        | 1        | 0     |
| j-type`101`  | `111` | 0     | 0               | 0        | 0        | 1     |

- **WBReg** (Write-Back Enable): habilita gravação no Banco de Registradores.  
- **ALUSrc** (MuxULA): 0 = operando B vem de registrador, 1 = vem de imediato/offset.  
- **dado-breg**: 1 = dado de memória vai para write-back; 0 = resultado da ALU.  
- **signal-control**: habilita gravação na RAM (`STR`).  
- **PCSel**: 1 = PC ← (PC+1) + signext(off3) (branch); 0 = PC ← PC + 1.  


### 2.4 Banco de Registradores (RegFile)  
- **Tipo:** 4 × 12 bits, 2 portas de leitura, 1 porta de escrita  
- **Função:**  
  - Lê `R[rs]`, `R[rt]`  
  - Escreve resultado em `R[rd]` (quando `WBReg=1`)

### 2.5 Unidade de Extensão de Sinal (Sign-Extend)  
- **Tipo:** Combinacional  
- **Função:** Expande `imm3` ou `off3` (3 bits) para 12 bits com cópia do bit de sinal.

### 2.6 MUX de Operandos da ALU (MuxULA)  
- **Tipo:** Multiplexador 2→1, 12 bits  
- **Função:** Seleciona entre `R[rt]` (operando reg-reg) ou `signext(imm3)` (I-type).

### 2.7 Unidade Lógica e Aritmética (ALU)  
- **Largura:** 12 bits  
- **Operações:** ADD, SUB, AND, OR, XOR, SHL, SHR, comparador para BEQ/BNQ  
- **Entradas:**  
  - `A = R[rs]`  
  - `B = MuxULA_out`  
  - `OpULA[2..0]`  
- **Saída:**  
  - `ALU_result[11..0]`  
  - `Zero` flag (para branches)

### 2.8 MUX de Write-Back (MuxWB)  
- **Tipo:** Multiplexador 2→1, 12 bits  
- **Função:** Seleciona entre `ALU_result` e `DataMem_out` para escrever em `RegFile`.

### 2.9 Memória de Dados (RAM)  
- **Tipo:** RAM word-addressable, 12 bits por palavra  
- **Função:** Carrega ou armazena dados em `LD`/`STR`.  
- **Entradas:**  
  - `addr = ALU_result[?]`  
  - `writeData = R[rd or rt]`  
  - `MemRead`, `MemWrite`  
- **Saída:**  
  - `DataMem_out[11..0]`

### 2.10 Unidade de Cálculo de Next PC  
- **Componentes:**  
  - **Incrementador** (`PC + 1`)  
  - **Somador de branch** (`(PC+1) + signext(off3)`)  
  - **MUX** (`PCsel`): escolhe  `PC+1` ou target de branch

---

## 3. Listagem de Componentes

| Componente                   | Quantidade | Largura | Descrição                                      |
|:-----------------------------|:----------:|:-------:|:-----------------------------------------------|
| **PC**                       | 1          | 12 bits | Registrador de fluxo de instruções             |
| **ROM (Instr. Memory)**      | 1          | 12 bits | Armazena programa                              |
| **Control Unit**             | 1          | –       | Decodifica opcode/funct3/cond → sinais de controle |
| **RegFile**                  | 1          | 4×12 bits | 4 registradores, 2 portas de leitura, 1 escrita |
| **Sign-Extend**              | 1          | 3→12 bits | Extende offset/imediato                        |
| **MuxULA**                   | 1          | 12 bits | Seleção imediato ou registrador para ALU       |
| **ALU**                      | 1          | 12 bits | Operações aritm./lógicas e comparações         |
| **MuxWB**                    | 1          | 12 bits | Seleção ALU ou memória para write-back         |
| **RAM (Data Memory)**        | 1          | 12 bits | Memória de dados word-addressable              |
| **Incrementador PC+1**       | 1          | 12 bits | Soma constante “+1”                            |
| **Somador de Branch**        | 1          | 12 bits | Soma PC+1 + signext(off3)                      |
| **Mux de NextPC (PCsel)**    | 1          | 12 bits | Escolhe fluxo normal ou branch                 |
| **Clock**                    | 1          | –       | Gera bordas de subida para todos registros     |

---


