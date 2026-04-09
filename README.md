# 🖥️ Documentação da CPU — Digital Simulator

## Visão Geral

Esta CPU foi projetada no simulador **Digital** como extensão de uma atividade anterior de construção de ALU. Ela é capaz de buscar instruções de uma memória de programa, decodificar opcodes, buscar operandos de uma memória de dados e executar operações aritméticas/lógicas via ALU.

A CPU opera com palavras de **8 bits** e possui um conjunto simplificado de instruções de 3 bits de opcode.

---

## Arquitetura

```
         ┌──────────────────────────────────────────────┐
         │                   CPU                        │
         │                                              │
  clk ──►│  ┌──────────┐    ┌──────────────────────┐    │
         │  │  Program  │    │   Memória de         │   │
reset ──►│  │ Counter   ├───►│   Instruções (EEPROM)│   │
         │  │  (8 bits) │    │   "Opcode"           │   │
         │  └──────────┘    └──────────┬───────────┘    │
         │                             │ opcode[2:0]    │
         │         ┌───────────────────┘                │
         │         │                                    │
         │         ▼                                    │
         │  ┌─────────────┐   ┌──────────────────────┐  │
         │  │ Decodificador│   │  Memória de Dados    │ │
         │  │  de Opcode  │   │  (EEPROM) "valor"    │  │
         │  └──────┬──────┘   └──────────┬───────────┘  │
         │         │                     │              │
         │         ▼                     ▼              │
         │  ┌──────────────────────────────────────┐    │
         │  │              ALU.dig                 │    │
         │  └───────────┬──────────────────────────┘    │
         │              │                               │
         │         Ac (acumulador)   Mq (mult/quoc.)    │
         └──────────────────────────────────────────────┘
```

---

## Componentes Principais

| Componente | Tipo | Descrição |
|---|---|---|
| `Clock` | Clock | Sinal de clock em tempo real. Label: `clk` |
| `Counter` (8 bits) | Contador | **Program Counter (PC)** — endereça a memória de instruções |
| `EEPROMDualPort "Opcode"` | Memória ROM | **Memória de Programa** — armazena os opcodes das instruções |
| `EEPROMDualPort "valor"` | Memória ROM | **Memória de Dados** — armazena os operandos/valores |
| `Splitter` (8→3,5) | Lógica | Separa o endereço do PC em **opcode [2:0]** e **operando [4:0]** |
| `ALU.dig` | Subcomponente | ALU desenvolvida na atividade anterior |
| `T_FF` | Flip-Flop T | Controla a **fase de execução** (busca vs. execução) |
| `Comparator "stop"` | Comparador | Detecta opcode `7` (instrução **HALT**) e para o clock |
| `Comparator` (3 bits) | Comparador | Detecta opcode `6` (instrução **STORE**) |
| `Driver` (tri-state, 8 bits) | Buffer | Controla o barramento de dados com sinal `str` |
| `Not`, `And`, `Or` | Lógica | Lógica de controle do ciclo e dos sinais de habilitação |
| `In "reset"` | Entrada | Sinal de reset do PC |
| `Out "Ac"` | Saída | Saída do **Acumulador** (resultado da ALU), 8 bits |
| `Out "Mq"` | Saída | Saída do **Registrador MQ** (multiplicação/quociente), 8 bits |

---

## Sinais e Redes (Tunnels)

Os tunnels são usados para conectar sinais à distância no circuito sem fios cruzados, funcionando como redes nomeadas.

| Nome do Tunnel | Bits | Descrição |
|---|---|---|
| `clk` | 1 | Clock distribuído para todos os componentes síncronos |
| `out_PC` | 8 | Saída atual do Program Counter (endereço corrente) |
| `Opcode` | 3 | Opcode decodificado da instrução atual |
| `q` | 1 | Saída do Flip-Flop T — indica a fase do ciclo (busca/execução) |
| `str` | 1 | Sinal de **store** — habilita escrita na EEPROM e os drivers tri-state |
| `clk_alu` | 1 | Clock condicional enviado à ALU — ativo apenas durante a fase de execução |
| `stop_16` | 1 | Sinal de **HALT** — ativado quando o opcode é `7`, para a CPU |

---

## Memórias

### Memória de Instruções — `"Opcode"` (EEPROM)

- **Endereçamento:** 3 bits (8 posições)
- **Largura de dado:** 3 bits (opcode)
- **Conteúdo:** `6*0, 6, 7`
  - Posições 0–5: opcode `0` (NOP / operação padrão da ALU)
  - Posição 6: opcode `6` (STORE)
  - Posição 7: opcode `7` (HALT)

### Memória de Dados — `"valor"` (EEPROM)

- **Endereçamento:** 8 bits
- **Largura de dado:** 8 bits
- **Conteúdo (hex):** `00, 01, 02, 03, 04, 05, 00, 00, 08, 0B, 0C, 0D, 0E, 0E, 0F, 10`

| Endereço | Valor (dec) |
|---|---|
| 0x00 | 0 |
| 0x01 | 1 |
| 0x02 | 2 |
| 0x03 | 3 |
| 0x04 | 4 |
| 0x05 | 5 |
| 0x06 | 0 |
| 0x07 | 0 |
| 0x08 | 8 |
| 0x09 | 11 |
| 0x0A | 12 |
| 0x0B | 13 |
| 0x0C | 14 |
| 0x0D | 14 |
| 0x0E | 15 |
| 0x0F | 16 |

---

## Conjunto de Instruções (ISA)

Esta CPU possui um conjunto mínimo de 3 instruções codificadas em 3 bits de opcode:

| Opcode (bin) | Opcode (dec) | Mnemônico | Descrição |
|---|---|---|---|
| `000` a `101` | 0–5 | `NOP` / `ALU_OP` | Operação padrão — alimenta a ALU com o operando da memória de dados |
| `110` | 6 | `STORE` | Armazena o valor do Acumulador (Ac) na EEPROM de dados. O sinal `str` é ativado para habilitar a escrita e os drivers tri-state |
| `111` | 7 | `HALT` | Para o clock da CPU (**Clock Inhibit**). O comparador detecta opcode = 7 e ativa `stop_16`, bloqueando o avanço do PC |

> **Nota:** O opcode `STORE` (6) é implementado como funcionalidade demonstrativa — o circuito ativa o sinal de escrita, mas não persiste dados de forma permanente na simulação atual.

---

## Ciclo de Execução

O ciclo da CPU é controlado pelo **Flip-Flop T** (`T_FF`), que alterna entre dois estados a cada borda de clock:

```
Estado q=0 (BUSCA / FETCH):
  1. O PC aponta para o endereço atual
  2. A EEPROM "Opcode" fornece o opcode da instrução
  3. O Splitter separa os bits do endereço do PC
  4. A EEPROM "valor" lê o operando correspondente
  5. Os drivers tri-state estão desabilitados (str=0)

Estado q=1 (EXECUÇÃO / EXECUTE):
  1. O clock da ALU (clk_alu) é habilitado
  2. A ALU processa a operação com os dados buscados
  3. Se opcode=6 (STORE): str=1, dados são escritos na EEPROM
  4. Se opcode=7 (HALT): stop_16=1, clock é bloqueado pelo AND com NOT(stop_16)
  5. Ao final, o PC é incrementado para a próxima instrução
```

### Diagrama de Temporização Simplificado

```
clk:      ___‾‾‾___‾‾‾___‾‾‾___
q (T_FF): ___‾‾‾‾‾‾‾___‾‾‾‾‾‾‾
          |FETCH|EXEC|FETCH|EXEC
clk_alu:  _______‾‾‾_______‾‾‾
```

---

## Registradores de Saída

| Saída | Bits | Descrição |
|---|---|---|
| `Ac` | 8 bits | **Acumulador** — resultado principal da ALU |
| `Mq` | 8 bits | **MQ (Multiplier/Quotient)** — registrador auxiliar da ALU, utilizado em operações de multiplicação e divisão |

---

## Integração com a ALU

A CPU utiliza o componente `ALU.dig` como subcomponente externo. A ALU recebe:

- **Opcode** (`Opcode` tunnel, 3 bits) — seleciona a operação
- **Operando** — vindo da EEPROM "valor" via barramento de dados
- **Clock da ALU** (`clk_alu`) — sinal de clock condicionado à fase de execução
- **Entrada B** — constante `0x00` (8 bits) como segundo operando padrão

E produz como saída:
- **Ac** — resultado do acumulador (8 bits)
- **Mq** — registrador MQ (8 bits)

> Para detalhes completos da ALU (operações suportadas, entradas e saídas internas), consulte a documentação da ponderada da ALU.
