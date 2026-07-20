# Manual de Fiação — Preparação do Circuito

Guia prático para montar o quadro e puxar os cabos **antes** de energizar e gravar o ESP32. Os valores aqui (pinos, resistores, limiares) são os mesmos do [`caixa-dagua.yaml`](caixa-dagua.yaml) — se você mudar algo na instalação, mude nos dois lugares.

> Detalhamento de engenharia (divisores, BOM completo, regras de segurança do firmware) está no [Memorial Descritivo](MEMORIAL-DESCRITIVO.md). Este manual é o passo a passo de **montagem física**.

---

## 0. Antes de começar

- **Desligue o disjuntor geral** do circuito da bomba antes de tocar em qualquer fiação de 220 V. Trabalhe com o quadro todo desenergizado até o checklist da seção 7.
- **Separe fisicamente** a fiação de 220 V (disjuntor → PZEM → contator → bomba) da fiação de sinal (3.3 V/5 V: ESP32, YYNMOS-4, boias, botão). Use canaletas/trilhos diferentes dentro do quadro, ou pelo menos mantenha distância — evita ruído no ADC e risco de contato acidental.
- **Cores de fio recomendadas** (NBR 5410): terra verde-amarelo, neutro azul-claro, fase em preto/vermelho/marrom. Para os sinais DC (3.3 V/5 V) use cores que não sejam essas três, para não confundir com a parte de potência.
- **Bornes de parafuso + terminais** (forquilha ou pino) em toda conexão — projeto é "zero solda" (trilho DIN).
- Ferramentas: multímetro (essencial — várias etapas abaixo pedem medição antes de energizar), chave de fenda/Philips para bornes, alicate de crimpar terminais, ferramenta de crimpagem RJ45 (se for crimpar o UTP você mesmo).

---

## 1. Visão geral da topologia

```
                         ┌─────────────── QUADRO DE COMANDO ───────────────┐
REDE 220V ──[disjuntor]──┤                                                 │
                         │   L ──► PZEM-004T (IN) ──► PZEM-004T (OUT) ──┐  │
                         │   N ──────────────────────────────────────┐ │  │
                         │                                           │ │  │
                         │                              CONTATOR ◄───┘ │  │
                         │                            (bobina A1/A2)   │  │
                         │                                 ▲           │  │
                         │                    RELÉ 5V ─────┘           │  │
                         │                       ▲                     │  │
                         │           YYNMOS-4 canal 4 (IN4)             │  │
                         │                       ▲                     │  │
                         │     ESP32-C3 ─── GPIO7 (rele)               │  │
                         │        │  │  │                              │  │
                         │  GPIO4 │  │  │ GPIO6      GPIO0 (ADC boias) │  │
                         │  GPIO5 │  │  │ GPIO21/20  GPIO1 (botão)     │  │
                         │        ▼  ▼  ▼   (UART PZEM)                │  │
                         │  YYNMOS-4 canais 1-3 (R/G/B)                │  │
                         │        │  │  │                              │  │
                         └────────┼──┼──┼──────────────────────────────┼──┘
                                  │  │  │                               │
                          UTP (6 vias)                            L' ──┴──► BOMBA (fase)
                                  │                                N' ─────► BOMBA (neutro)
                                  ▼
                    COZINHA: botão inox + LED RGB anodo comum

                    CAIXA SUPERIOR: 2 boias reed + R 1k/4.7k/10k(EOL)
                    2 fios ────────────────────────► GPIO0 (com pull-up 3.3k
                                                       no quadro + cap 100nF)
```

- **PZEM-004T** mede a corrente **em série** na fase (L) — a fase passa por dentro do módulo (bornes IN→OUT), não é grampo externo (shunt interno).
- **YYNMOS-4** é uma chave *low-side* (liga pelo lado do GND): sinal ALTO do ESP32 no canal correspondente fecha aquele canal para o GND. Por isso a lógica de wiring do LED usa **anodo comum em +5V fixo** e cada cor é puxada para baixo pelo canal.
- O **relé** só existe porque o YYNMOS-4 sozinho não chaveia 220 V — ele aciona o relé de baixa potência, que por sua vez aciona a **bobina do contator**, que é quem efetivamente liga a bomba.

---

## 2. Potência — Disjuntor → PZEM → Contator → Bomba

| Etapa | De | Para | Observação |
|---|---|---|---|
| 1 | Rede 220 V | Disjuntor termomagnético 6–10 A curva C | Dedicado só ao circuito da bomba |
| 2 | Disjuntor (fase, L) | PZEM-004T, borne **IN** (L) | Neutro (N) também entra no PZEM (borne IN N) — o módulo referencia tensão entre L e N |
| 3 | PZEM-004T, borne **OUT** (L) | Contator, terminal de entrada (linha) | A corrente medida é a que passa por este trecho — é o que o firmware usa pra E6 (rotor preso) / E7 (bomba a seco) |
| 4 | PZEM-004T, borne **OUT** (N) | Segue direto até a bomba | Neutro geralmente **não** passa pelo contator monofásico (só a fase é chaveada) — confirme quantos polos tem o seu contator antes de fiar |
| 5 | Contator, terminal de saída (linha) | Bomba, fase | Bomba Intech BSCA2 1/4 HP, 220 V |
| — | Neutro (do passo 4) | Bomba, neutro | |

**Bobina do contator (A1/A2):**
- A1/A2 são acionados pelo **módulo relé** (contato NA do relé em série com a alimentação 220 V da bobina).
- Instale o **supressor de surto** (snubber RC ou varistor 275 V) **em paralelo direto nos bornes A1/A2** — protege o contato do relé do arco de abertura toda vez que a bobina desliga.

**Módulo relé:**
- Alimentação: 5 V (fonte DIN) + GND comum com o ESP32.
- Sinal (IN): vem do **canal 4 do YYNMOS-4**.
- Contato de potência (COM/NA): em série com a fase que alimenta a bobina do contator (passo acima).

> ⚠️ **Antes de energizar de vez:** com o disjuntor desligado, meça continuidade da bobina do contator (deve ter uma resistência baixa e finita — nunca aberta nem curto) e confira visualmente que L e N não estão trocados nos bornes do PZEM.

---

## 3. Sinal — ESP32-C3 e pinagem

Tabela oficial de pinos (igual ao `substitutions:` do YAML — se mudar aqui, mude lá também):

| Sinal | GPIO (ESP32-C3) | Vai para | Direção |
|---|---|---|---|
| `pino_adc_boias` | GPIO0 (ADC1) | Nó do divisor das boias | Entrada analógica |
| `pino_botao` | GPIO1 | Contato seco do botão (via UTP) | Entrada digital (pull-up interno, invertido) |
| `pino_led_r` | GPIO4 | YYNMOS-4, canal 1 (IN1) | Saída PWM |
| `pino_led_g` | GPIO5 | YYNMOS-4, canal 2 (IN2) | Saída PWM |
| `pino_led_b` | GPIO6 | YYNMOS-4, canal 3 (IN3) | Saída PWM |
| `pino_rele` | GPIO7 | YYNMOS-4, canal 4 (IN4) → módulo relé | Saída digital |
| `pino_uart_tx` | GPIO21 | PZEM-004T, **RX** | Saída serial |
| `pino_uart_rx` | GPIO20 | PZEM-004T, **TX** | Entrada serial |

> TX do ESP32 vai no RX do PZEM e vice-versa — é cruzado, como qualquer UART ponto-a-ponto.

**YYNMOS-4 — alimentação:**
- VCC: 5 V (fonte DIN)
- GND: comum com o ESP32, a fonte 5V e o relé
- IN1–IN4: GPIO4, GPIO5, GPIO6, GPIO7 do ESP32 (nível 3.3 V — o módulo aceita)
- OUT1–OUT3: cátodos R/G/B do LED da cozinha
- OUT4: entrada de sinal do módulo relé

**PZEM-004T — alimentação e sinal (além dos bornes de potência da seção 2):**
- VCC: 5 V (fonte DIN)
- GND: comum
- RX ← GPIO21 (`pino_uart_tx`)
- TX → GPIO20 (`pino_uart_rx`)

> Confirme no datasheet do **seu** módulo PZEM-004T V3 se ele tem isolamento galvânico entre o lado AC (L/N/shunt) e o lado lógico (5V/GND/RX/TX). Mesmo que tenha, trate o módulo inteiro como parte do circuito de 220 V ao manuseá-lo — nunca toque nos bornes RX/TX/5V com o disjuntor ligado.

---

## 4. Caixa superior — boias (2 fios, laço supervisionado)

Monte **na caixa**, perto das boias, o seguinte arranjo em **paralelo**, descendo só 2 fios até o quadro:

| Componente | Ligação |
|---|---|
| Boia **Mínimo** (reed switch) | Em série com resistor **1 kΩ** |
| Boia **Máximo** (reed switch) | Em série com resistor **4.7 kΩ** |
| Resistor **10 kΩ** (fim de linha / EOL) | Direto em paralelo com as duas boias (é este resistor que torna o rompimento do cabo detectável — **instale-o na caixa, não no quadro**) |

Os três ramos ficam em paralelo entre os **2 fios** que descem ao quadro (chame de fio "A" e fio "B" — não há polaridade entre eles, mas mantenha consistência com o outro lado).

**No quadro**, os 2 fios chegam assim:
- Fio A → nó do ADC (GPIO0), que também recebe:
  - **Resistor pull-up de 3.3 kΩ** até 3.3 V
  - **Capacitor cerâmico 100 nF** até GND (filtro de ruído)
- Fio B → GND (mesmo GND do ESP32)

**Boias fecham contato quando a água está ABAIXO delas.** Tensões esperadas no ADC (para conferir com multímetro antes de plugar no ESP32):

| Situação | Resistência do laço | Tensão no nó ADC |
|---|---|---|
| Cabo rompido / boia desconectada | ∞ | ~3.3 V |
| Ambas abertas (caixa cheia) | 10 kΩ | ~2.48 V |
| Só a máxima fechada (intermediário) | 4.7k ∥ 10k ≈ 3.2 kΩ | ~1.62 V |
| Ambas fechadas (crítico) | 1k ∥ 4.7k ∥ 10k ≈ 762 Ω | ~0.62 V |
| Cabo em curto | ~0 | ~0 V |

> Meça essas tensões com um multímetro **antes** de conectar o fio A ao GPIO0 — se bater com a tabela, a fiação das boias está correta.

---

## 5. Cozinha — botão + LED (cabo UTP)

Um único cabo de rede (Cat5e/Cat6) liga o quadro à cozinha, usando **6 das 8 vias**:

| Via (UTP) | Função | Vai para (no quadro) |
|---|---|---|
| 1 | 5 V constante (anodo comum do LED) | Fonte DIN 5 V (direto, **não** passa pelo YYNMOS-4) |
| 2 | Cátodo R | YYNMOS-4 OUT1 |
| 3 | Cátodo G | YYNMOS-4 OUT2 |
| 4 | Cátodo B | YYNMOS-4 OUT3 |
| 5 | Contato seco do botão — GND | GND comum |
| 6 | Contato seco do botão — sinal | GPIO1 do ESP32 |
| 7, 8 | Não usadas | — |

- Botão: pulsador antivandalismo inox 19–22 mm, **momentâneo** (sem trava), LED RGB integrado **anodo comum**, 3–6 V.
- Use conectores RJ45 fêmea/macho nas duas pontas (quadro e cozinha) ou emende direto nos bornes — mas mantenha o par de fios do botão (vias 5/6) junto (par trançado), para reduzir ruído no clique.
- Confirme com multímetro, no botão isolado (sem o cabo ainda ligado ao quadro), que pressionar fecha o contato entre as vias 5 e 6 e que a resistência do LED entre a via 1 e cada uma das vias 2/3/4 é a de um LED direto (não invertido) — evita descobrir "anodo comum" errado só depois de tudo montado.

---

## 6. Fonte e alimentação geral

| Item | Alimenta |
|---|---|
| Fonte chaveada trilho DIN 5 V / 2 A | ESP32-C3 (via 5V→3.3V do próprio módulo, ou pino 5V se disponível), YYNMOS-4, módulo relé, PZEM-004T (lado lógico), LED da cozinha (via anodo) |

Todo o GND de baixa tensão (ESP32, YYNMOS-4, relé, PZEM lógico, boias, botão) deve ser **um único nó comum**. Não misture esse GND com o neutro (N) do lado de 220 V.

---

## 7. Checklist antes de energizar

Com o disjuntor **ainda desligado**:

1. [ ] Continuidade da bobina do contator (A1/A2) — resistência baixa e finita.
2. [ ] Tensões do divisor das boias batem com a tabela da seção 4 (medir com multímetro alimentando o nó ADC com 3.3 V de uma fonte de bancada, ou já com o ESP32 ligado via USB antes de energizar a bomba).
3. [ ] Botão: contato seco fecha/abre corretamente entre as vias 5/6 do UTP.
4. [ ] LED: anodo comum confirmado (via 1 = +5V, vias 2/3/4 = cátodos R/G/B).
5. [ ] Fase e neutro não estão trocados nos bornes IN do PZEM-004T.
6. [ ] Snubber/varistor instalado nos bornes A1/A2 do contator.
7. [ ] Fiação de 220 V fisicamente separada da fiação de sinal dentro do quadro.
8. [ ] GND comum de todo o lado de baixa tensão é um nó único, sem contato com o neutro de 220 V.

Depois disso:

9. [ ] Grave o firmware via USB (`esphome run caixa-dagua.yaml --device /dev/cu.usbmodemXXXX`) **antes** de energizar a bomba pela primeira vez — o ESP32 precisa estar rodando a FSM (E9 Boot Seguro) antes do disjuntor ir para ON.
10. [ ] Ligue o disjuntor. Verifique no log/Home Assistant: tensão das boias, estado da FSM (deve entrar em E0 Standby ou E1 conforme o nível), tensão de rede e corrente no PZEM.
11. [ ] Use o botão **"Testar Bomba (Medir Corrente)"** do Home Assistant (roda a bomba por 10 s a partir do Standby) para calibrar os limiares `Corrente Máxima` e `Corrente Mínima` (menu de números do dispositivo) com a corrente real medida, em vez de confiar só nos defaults de fábrica (3.0 A / 0.5 A).

---

## 8. Referência rápida — BOM

Ver a lista completa com quantidades e observações no [Memorial Descritivo, seção 5](MEMORIAL-DESCRITIVO.md#5-bill-of-materials-bom-definitivo). Resumo do que este manual usa:

- Bomba Intech BSCA2 1/4 HP 220 V
- 2× boia reed switch + resistores 1 kΩ / 4.7 kΩ / 10 kΩ (EOL)
- Disjuntor 6–10 A curva C
- Fonte trilho DIN 5 V/2 A
- ESP32-C3 + YYNMOS-4 (4 canais)
- Módulo relé 5 V 1 canal + Contator monofásico 12 A AC-3 (bobina 220 V)
- PZEM-004T 10 A (shunt interno)
- Capacitor cerâmico 100 nF + resistor pull-up 3.3 kΩ
- Supressor de surto (snubber RC ou varistor 275 V)
- Botão inox antivandalismo com LED RGB anodo comum + cabo UTP Cat5e/6
