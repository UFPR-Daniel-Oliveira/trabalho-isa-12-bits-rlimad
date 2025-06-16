## Descritivo de Codificação do Loop de Duas Fases

A seguir mostramos, instrução a instrução, como transformar cada linha de assembly em sua forma binária de 12 bits e em hexadecimal, partindo do pressuposto de explorarmos a capacidade da ISA, por meio das operações de adição e jump. 

---

### 1. `ADDI R3, R3, +3`

- **Montagem**: `ADDI R3, R3, 3`  
- **Formato**: I-type  
- **Campos**  
  - **opcode** (bits 11–9): `001` (ADDI)  
  - **imm3**   (bits  8–6): `011` (+3)  
  - **rd**     (bits  5–4): `11` (R3)  
  - **rs**     (bits  3–2): `11` (R3)  
  - **rt**     (bits  1–0): `00` (pad)  
- **Concatenação**:  
  001 | 011 | 11 | 11 | 00
- **Binário (12 bits)**: `001011111100`  
- **Hexadecimal**: `0x2FC`  

---

### 2. `STR R1, [R2 + 0]`

- **Montagem**: `STR R1, [R2, 0]`  
- **Formato**: I-type (store)  
- **Campos**  
- **opcode** (11–9): `100` (STR)  
- **imm3**   ( 8–6): `000` (offset 0)  
- **rd**     ( 5–4): `00` (inativo em STR)  
- **rs**     ( 3–2): `10` (R2)  
- **rt**     ( 1–0): `01` (R1)  
- **Concatenação**:  
100 | 000 | 00 | 10 | 01
- **Binário**: `100000001001`  
- **Hexadecimal**: `0x809`  

---

### 3. `ADDI R1, R1, +1`  (`r1++`)

- **Montagem**: `ADDI R1, R1, 1`  
- **Formato**: I-type  
- **Campos**  
- **opcode** (11–9): `001`  
- **imm3**   ( 8–6): `001` (+1)  
- **rd**     ( 5–4): `01` (R1)  
- **rs**     ( 3–2): `01` (R1)  
- **rt**     ( 1–0): `00` (pad)  
- **Concatenação**:  
001 | 001 | 01 | 01 | 00
- **Binário**: `001001010100`  
- **Hexadecimal**: `0x254`  

---

### 4. `ADDI R2, R2, +1`  (`r2++`)

- **Montagem**: `ADDI R2, R2, 1`  
- **Formato**: I-type  
- **Campos**  
- **opcode** (11–9): `001`  
- **imm3**   ( 8–6): `001`  
- **rd**     ( 5–4): `10` (R2)  
- **rs**     ( 3–2): `10` (R2)  
- **rt**     ( 1–0): `00`  
- **Concatenação**:  
001 | 001 | 10 | 10 | 00
- **Binário**: `001001101000`  
- **Hexadecimal**: `0x268`  

---

### 5. `BNQ R1, R3, –2` (loop curto)

- **Montagem**: `BNQ R1, R3, –2`  
- **Formato**: J-type curto  
- **Campos**  
- **opcode** (11–9): `101` (branch curto)  
- **off3**   ( 8–6): `110` (–2 em dois-complemento)  
- **cond**   (   5): `1` - BNQ  
- **pad**    (   4): `0`  
- **rs**     ( 3–2): `01` (R1)  
- **rt**     ( 1–0): `11` (R3)  
- **Concatenação**:  
101 | 110 | 1 | 0 | 01 | 11
- **Binário**: `101101100111`  
- **Hexadecimal**: `0xB67`  

---

> **Observação:**  
> Após a quinta instrução, o PC já avançou (`PC+1`) e, como o offset é –2, o novo PC = `(PC+1) – 2` → volta duas instruções, repetindo o corpo do loop.

---

### Estrutura final em memória

| Endereço | Conteúdo binário | Conteúdo hex |
|:--------:|:----------------:|:------------:|
| 0        | 001011111100     | 0x2FC       |
| 1        | 100000001001     | 0x809       |
| 2        | 001001010100     | 0x254       |
| 3        | 001001101000     | 0x268       |
| 4        | 101101100111     | 0xB67       |
| 5        | 001011111100     | 0x2FC       |
| 6        | 100000001001     | 0x809       |
| 7        | 001001010100     | 0x254       |
| 8        | 001001101000     | 0x268       |
| 9        | 101101100111     | 0xB67       |

### Código em C

```c
  int main() {
    int a[3];
    int b[3];
    int i = 0;
    int contador = 3;

    // Loop A: preenche a[0..2]
    for (i; i < contador; i++)
        a[i] = i;

    contador += 3;

    // Loop B: preenche b[0..2]
    for (int k = 0; i < contador; k++, i++)
        b[k] = i;

    return 0;
}

```

> **Observação:**  
> Para fins demonstrativos, o endereço lógico de B é contiguo ao endereço de A

### Código em Assembly

```asm
    ; Inicializa vetor A
    ADDI R3, R3, 3           ; R3 = 3
loopA:
    STR  R1, [R2 + 0]        ; MEM[R2] = R1
    ADDI R1, R1, 1           ; R1++
    ADDI R2, R2, 1           ; R2++
    BNQ  R1, R3, -2          ; if (R1 != R3) - loopA

    ; Inicializa vetor B (contador += 3)
    ADDI R3, R3, 3           ; R3 += 3
loopB:
    STR  R1, [R2 + 0]        ; MEM[R2] = R1
    ADDI R1, R1, 1           ; R1++
    ADDI R2, R2, 1           ; R2++
    BNQ  R1, R3, -2          ; if (R1 != R3) - loopB
```

### Código Binário 

```bin
  001011111100 -> addi r3, 3
  100000001001 -> store r1 em endereco(r2)
  001001010100 -> r1++
  001001101000 -> r2++
  101101100111 -> BNQ r1,r3
  001011111100 -> addi r3, 3
  100000001001 -> store r1 em endereco(r2)
  001001010100 -> r1++
  001001101000 -> r2++
  101101100111 -> BNQ r1,r3
```

### Código Hex 

```hex
  0x2FC
  0x809
  0x254
  0x268
  0xB67
  0x2FC
  0x809
  0x254
  0x268
  0xB67
```
