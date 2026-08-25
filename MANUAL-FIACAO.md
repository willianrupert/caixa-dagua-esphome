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
                         │     ESP32-S3 ─── GPIO21 (rele)              │  │
                         │        │  │  │                              │  │
                         │ GPIO16 │  │  │ GPIO18     GPIO4 (ADC boias) │  │
                         │ GPIO17 │  │  │ GPIO9/10   GPIO5 (botão)     │  │
                         │        ▼  ▼  ▼   (UART PZEM)                │  │
                         │  YYNMOS-4 pwm1-3 (R/G/B)                    │  │
                         │        │  │  │                              │  │
                         └────────┼──┼──┼──────────────────────────────┼──┘
                                  │  │  │                               │
                          UTP (8 vias)                            L' ──┴──► BOMBA (fase)
                                  │                                N' ─────► BOMBA (neutro)
                                  ▼
                    COZINHA: botão inox + LED RGB anodo comum
                             + microswitch da porta de madeira

                    CAIXA SUPERIOR: 2 boias reed + R 1k/4.7k/10k(EOL)
                    2 fios ────────────────────────► GPIO4 (com pull-up 3.3k
                                                       no quadro + cap 100nF)

                    GRADE PRINCIPAL: 2 microswitches em série na fechadura
                    cabo 2 vias ─────────────────────► GPIO6 (seção 9)

                    GRADE SECUNDÁRIA: 2 microswitches em série na fechadura
                    cabo 2 vias ─────────────────────► GPIO7 (seção 9)
```

- **PZEM-004T** mede a corrente **em série** na fase (L) — a fase passa por dentro do módulo (bornes IN→OUT), não é grampo externo (shunt interno).
- **YYNMOS-4** é uma chave *low-side* (liga pelo lado do GND): sinal ALTO do ESP32 no canal correspondente fecha aquele canal para o GND. Por isso a lógica de wiring do LED usa **anodo comum em +5V fixo** e cada cor é puxada para baixo pelo canal. Só os canais 1–3 são usados (LED RGB) — o canal 4 não é mais usado.
- O **relé** é um módulo pronto 5 V ligado **direto no GPIO21 do ESP32** (sem passar pelo YYNMOS-4), porque o ESP32 sozinho não chaveia 220 V — ele aciona o relé de baixa potência, que por sua vez aciona a **bobina do contator**, que é quem efetivamente liga a bomba. O módulo é *ativo em HIGH* (trigger jumper em "H"): o GPIO21 em nível ALTO liga o relé diretamente, sem inversão no ESPHome.

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
- Alimentação: VCC 5 V (fonte DIN) + GND comum com o ESP32. Se o módulo tiver jumper **JD-VCC** (optoacoplador), mantenha-o ligado ao VCC — o projeto usa uma única fonte 5 V para toda a baixa tensão (ver seção 6); isolamento galvânico de verdade exigiria uma segunda fonte dedicada só ao JD-VCC, o que não faz parte deste BOM.
- Sinal (IN): vem **direto do GPIO21 do ESP32** (nível 3.3 V — o módulo aceita). Confira o jumper de trigger do módulo: este projeto assume **ativo em HIGH** (jumper "H", sem `inverted` no ESPHome). Se o seu módulo estiver no modo "L" (ativo em LOW), troque o jumper para "H" ou adicione `inverted: true` no `pin:` do YAML — mas teste com multímetro/LED antes de ligar o contator (item 9 da seção 7).
- Contato de potência (COM/NA): em série com a fase que alimenta a bobina do contator (passo acima).

> ⚠️ **Antes de energizar de vez:** com o disjuntor desligado, meça continuidade da bobina do contator (deve ter uma resistência baixa e finita — nunca aberta nem curto) e confira visualmente que L e N não estão trocados nos bornes do PZEM.

---

## 3. Sinal — ESP32-S3 e pinagem

Tabela oficial de pinos (igual ao `substitutions:` do YAML — se mudar aqui, mude lá também):

| Sinal | GPIO (ESP32-S3) | Vai para | Direção |
|---|---|---|---|
| `pino_adc_boias` | GPIO4 (ADC1_CH3) | Nó do divisor das boias | Entrada analógica |
| `pino_botao` | GPIO5 | Contato seco do botão (via UTP via 6) | Entrada digital (pull-up interno, invertido) |
| `pino_grade_principal` | GPIO6 | Grade principal — 2 microswitches em série | Entrada digital (pull-up interno) |
| `pino_grade_secundaria` | GPIO7 | Grade secundária — 2 microswitches em série | Entrada digital (pull-up interno) |
| `pino_porta_cozinha` | GPIO8 | Microswitch único, porta da cozinha (via UTP via 8) | Entrada digital (pull-up interno) |
| `pino_uart_tx` | GPIO9 | PZEM-004T, **RX** | Saída serial |
| `pino_uart_rx` | GPIO10 | PZEM-004T, **TX** | Entrada serial |
| `pino_led_r` | GPIO16 | YYNMOS-4, **pwm1** | Saída PWM |
| `pino_led_g` | GPIO17 | YYNMOS-4, **pwm2** | Saída PWM |
| `pino_led_b` | GPIO18 | YYNMOS-4, **pwm3** | Saída PWM |
| `pino_rele` | GPIO21 | Módulo relé 5V, pino **IN** (direto, sem passar pelo YYNMOS-4) | Saída digital, ativa em HIGH (sem `inverted`) |

### 3.1 Por que estes pinos, e não os do ESP32-C3

O projeto nasceu para ESP32-C3 e migrou para **S3** em 2026-08-25. Não foi só
trocar a placa no YAML: pino limpo no C3 não é necessariamente limpo no S3.

| Pino | Era, no C3 | Por que saiu no S3 |
|---|---|---|
| GPIO0 | ADC das boias | **Strapping de boot**, e ligado ao botão BOOT na maioria das placas |
| GPIO20 | UART RX do PZEM | **D+ do USB nativo** — conflitaria com a própria gravação (GPIO19 é o D−) |
| GPIO3 | Grade principal | **Strapping** (seleção de JTAG) |
| GPIO8 | Grade secundária | Nada — no S3 ele é **limpo**. Aqui o S3 melhora, e a ressalva "não testado no boot" que o C3 exigia deixou de existir |

Também evitados por princípio: **GPIO26–37** (flash/PSRAM do módulo),
**GPIO43/44** (UART0 da ponte USB-serial), **GPIO45/46** (strapping) e
**GPIO48** (LED RGB embutido em boa parte das placas S3).

**O ADC tem que ficar entre GPIO1 e GPIO10.** No S3 só o ADC1 funciona com o
Wi-Fi ligado — o ADC2 é disputado com o rádio e devolve leitura instável. É
por isso que as boias foram para GPIO4 e não para um pino alto qualquer.

> ⚠️ **Confira no silk da sua placa que os 11 pinos existem nos headers**
> antes de soldar. Placas S3 genéricas variam no que trazem para fora — se
> algum não estiver disponível, trocar é uma linha no `substitutions:` do
> YAML, mas é muito mais barato descobrir agora que depois de crimpar.

> TX do ESP32 vai no RX do PZEM e vice-versa — é cruzado, como qualquer UART ponto-a-ponto.
>
> Os 3 pinos de porta usam a mesma convenção de contato seco do botão
> (pull-up interno, fecha para GND), mas **sem** `inverted` — polaridade
> deliberadamente oposta à do botão. Explicação completa na seção 9.

**YYNMOS-4 — alimentação:**
- VCC: 5 V (fonte DIN)
- GND: comum com o ESP32, a fonte 5V e o relé
- pwm1–pwm3: GPIO16, GPIO17, GPIO18 do ESP32 (nível 3.3 V — o módulo aceita). Canal 4 (IN4/OUT4) não é usado.
- OUT1–OUT3: cátodos R/G/B do LED da cozinha

**PZEM-004T — alimentação e sinal (além dos bornes de potência da seção 2):**
- VCC: 5 V (fonte DIN)
- GND: comum
- RX ← GPIO9 (`pino_uart_tx`)
- TX → GPIO10 (`pino_uart_rx`)

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
- Fio A → nó do ADC (GPIO4), que também recebe:
  - **Resistor pull-up de 3.3 kΩ** até 3.3 V
  - **Capacitor cerâmico 100 nF** até GND (filtro de ruído)
- Fio B → GND (mesmo GND do ESP32)

**As duas boias fecham contato quando a água está ABAIXO delas.**

> ⚠️ **A orientação das boias não se inverte** — nem "só a de cima", nem para
> tratar transbordo. Já foi tentado (2026-08-25) e quebra duas coisas de uma
> vez: "cheia" deixa de ser a ausência de sinal (aí um reed morto ou fio
> rompido passa a ler "ainda não encheu" e a bomba não desliga), e dois
> estados normais colapsam para ~90 mV de diferença. O porquê completo está
> no [Memorial 4.1.1](MEMORIAL-DESCRITIVO.md#411-por-que-as-duas-boias-fecham-secas--e-por-que-não-se-gira).
> Vale também na **reposição**: boia nova entra na mesma orientação da que
> saiu, senão a leitura inteira muda de significado.

Tensões esperadas no ADC (para conferir com multímetro antes de plugar no ESP32):

| Situação | Resistência do laço | Tensão no nó ADC |
|---|---|---|
| Cabo rompido / boia desconectada | ∞ | ~3.3 V |
| Ambas abertas (caixa cheia) | 10 kΩ | ~2.48 V |
| Só a máxima fechada (intermediário) | 4.7k ∥ 10k ≈ 3.2 kΩ | ~1.62 V |
| Ambas fechadas (crítico) | 1k ∥ 4.7k ∥ 10k ≈ 762 Ω | ~0.62 V |
| Cabo em curto | ~0 | ~0 V |

> Meça essas tensões com um multímetro **antes** de conectar o fio A ao GPIO4 — se bater com a tabela, a fiação das boias está correta. Teste rápido de orientação, fácil de fazer sozinho: com o nível **entre** as boias, o laço deve medir **~3,2 kΩ**. Se medir ~10 kΩ ou ~909 Ω, alguma boia está girada.

### 4.1 Técnica de campo — caixinha de resina PU40

Registro da montagem usada na caixa do 2º andar (2026-08-22), porque funciona
e vale repetir nas próximas caixas: os três ramos (boia mín. + 1kΩ, boia máx.
+ 4.7kΩ, EOL 10kΩ) ficam soldados dentro de uma caixinha, depois preenchida
com resina PU40 — sela contra umidade sem exigir caixa estanque de verdade.

**Dois nós, três ramos, tudo em paralelo entre eles:**

```
LADO DOS SENSORES                                 LADO DO CABO
      (Ficam na Caixa D'água)                           (Desce até o ESP32)

                     _______________________________________
                    |           CAIXINHA DE RESINA          |
                    |                                       |
 [BOIA MÁXIMA]      |                                       |
   Fio Máx 1 -------|--------+                              |
                    |        |                              |
   Fio Máx 2 -------|---[ Resistor 4.7kΩ ]---+              |
                    |        |               |              |
                    |      [ * ] NÓ 1      [ * ] NÓ 2       |
                    |        |               |              |
 [BOIA MÍNIMA]      |        |               |              |
   Fio Mín 1 -------|--------+------------------------------|---> Fio COMUM
                    |                        |              |    (vai pro GND do quadro,
   Fio Mín 2 -------|---[ Resistor 1.0kΩ ]---+              |     NUNCA 3.3V — o pull-up
                    |        |               |              |     já mora no quadro)
                    |    [ Resistor 10kΩ, EOL ]--------------|---> Fio de SINAL
                    |        |               |              |    (vai pro ADC/GPIO4
                    |________+_______________+_______________|     do quadro)
                                 (Preenchido com PU40)
```

- **NÓ 1** (comum): Fio Máx 1 + Fio Mín 1 + **uma perna do resistor EOL 10kΩ**
  + Fio COMUM que desce ao quadro. Torça os quatro juntos, uma gota de solda.
- **NÓ 2** (sinal): pernas de trás dos resistores 4.7kΩ e 1.0kΩ + **a outra
  perna do EOL 10kΩ** + Fio de SINAL que desce ao quadro.
- **Trilhas isoladas:** Fio Máx 2 solda direto na perna da frente do 4.7kΩ;
  Fio Mín 2 solda direto na perna da frente do 1.0kΩ.
- **Fio COMUM vai pro GND do quadro, nunca pro 3.3V** — o pull-up de 3.3kΩ
  que fecha o divisor de tensão já mora no quadro, no nó do ADC (ver tabela
  acima). Se o comum fosse pro 3.3V também, não sobraria caminho pro GND e o
  ADC leria sempre perto de 3.3V, não importa a posição das boias.

> ⚠️ **O resistor EOL de 10kΩ é o terceiro ramo, não um extra opcional** — sem
> ele, "ambas as boias abertas" (caixa cheia de verdade) e "cabo rompido"
> ficam **eletricamente idênticos** (os dois viram circuito aberto entre os
> nós, ~3.3V no ADC). É exatamente o caso que o laço supervisionado existe
> pra distinguir (seção 4, tabela de tensões) — sem o EOL, a caixa cheia
> seria diagnosticada como sensor com defeito, nunca como "cheia" de
> verdade. **Instale-o na caixa** (NÓ 1 ↔ NÓ 2), nunca no quadro.

> ✅ **EOL corrigido e medido em campo (2026-08-25):** com água entre as
> boias, o laço mediu **3,19 kΩ no painel** — batendo com 4.7k ∥ 10k ≈ 3,2 kΩ
> e confirmando que o terceiro ramo entrou no circuito. Os resistores do lote
> usado ficaram bem dentro da tolerância (~0,3% do nominal nessa combinação).

**Dica de campo — proteja os resistores antes da resina:** as pernas cruas
dos resistores ficam vulneráveis a encostar no NÓ vizinho quando a caixinha é
apertada pra caber. Vista cada resistor com um pedaço de espaguete
termorretrátil (ou fita isolante), deixando só as pontas expostas pra solda —
cria um túnel protetor e evita curto acidental depois que já não dá mais pra
abrir e corrigir.

---

## 5. Cozinha — botão + LED (cabo UTP)

Um único cabo de rede (Cat5e/Cat6) liga o quadro à cozinha, usando as **8 vias**:

| Via (UTP) | Função | Vai para (no quadro) |
|---|---|---|
| 1 | 5 V constante (anodo comum do LED) | Fonte DIN 5 V (direto, **não** passa pelo YYNMOS-4) |
| 2 | Cátodo R | YYNMOS-4 OUT1 |
| 3 | Cátodo G | YYNMOS-4 OUT2 |
| 4 | Cátodo B | YYNMOS-4 OUT3 |
| 5 | Contato seco do botão — GND | GND comum |
| 6 | Contato seco do botão — sinal | GPIO5 do ESP32 (`pino_botao`) |
| 7 | Contato seco da porta da cozinha — GND | GND comum |
| 8 | Contato seco da porta da cozinha — sinal | GPIO8 do ESP32 (`pino_porta_cozinha`) |

- Botão: pulsador antivandalismo inox 19–22 mm, **momentâneo** (sem trava), LED RGB integrado **anodo comum**, 3–6 V.
- Use conectores RJ45 fêmea/macho nas duas pontas (quadro e cozinha) ou emende direto nos bornes — mas mantenha o par de fios do botão (vias 5/6) junto (par trançado), para reduzir ruído no clique.
- Confirme com multímetro, no botão isolado (sem o cabo ainda ligado ao quadro), que pressionar fecha o contato entre as vias 5 e 6 e que a resistência do LED entre a via 1 e cada uma das vias 2/3/4 é a de um LED direto (não invertido) — evita descobrir "anodo comum" errado só depois de tudo montado.

---

## 6. Fonte e alimentação geral

| Item | Alimenta |
|---|---|
| Fonte Hi-Link 5 V / 1 A (módulo AC-DC, não trilho DIN) | ESP32-S3 (via 5V→3.3V do próprio módulo, ou pino 5V se disponível), YYNMOS-4, módulo relé, PZEM-004T (lado lógico), LED da cozinha (via anodo) |

Todo o GND de baixa tensão (ESP32, YYNMOS-4, relé, PZEM lógico, boias, botão) deve ser **um único nó comum**. Não misture esse GND com o neutro (N) do lado de 220 V.

> ⚠️ **Orçamento de corrente mais apertado que a fonte original (2A → 1A).**
> Estimativa de consumo simultâneo no pior caso: ESP32-S3 com Wi-Fi ativo
> (~100–150 mA em regime, picos de TX bem mais altos por microssegundos,
> absorvidos pela capacitância da própria placa) + relé energizado (~80 mA)
> + LED da cozinha em branco pleno (~60 mA) + PZEM lógico (~30–50 mA) — soma
> em torno de 300–400 mA de regime, bem dentro de 1 A, mas sem a folga
> generosa que 2 A dava. Os microswitches das portas **não** entram nessa
> conta (pull-up interno do ESP32, consumo em µA). Meça a corrente real do
> trecho 5V com tudo ligado (relé acionado + Wi-Fi ativo) antes de dar por
> encerrado — mais um item pro checklist da seção 7.

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
9. [ ] Jumper de trigger do módulo relé confirmado em **"H" (ativo em HIGH)** — ou o YAML ajustado (`inverted: true`) se o seu módulo só tiver "L".

Depois disso:

10. [ ] Grave o firmware via USB (`esphome run caixa-dagua.yaml --device /dev/cu.usbmodemXXXX`) **antes** de energizar a bomba pela primeira vez — o ESP32 precisa estar rodando a FSM (E9 Boot Seguro) antes do disjuntor ir para ON.
11. [ ] Ligue o disjuntor. Verifique no log/Home Assistant: tensão das boias, estado da FSM (deve entrar em E0 Standby ou E1 conforme o nível), tensão de rede e corrente no PZEM.
12. [ ] Use o botão **"Testar Bomba (Medir Corrente)"** do Home Assistant (roda a bomba por 10 s a partir do Standby) para calibrar os limiares `Corrente Máxima` e `Corrente Mínima` (menu de números do dispositivo) com a corrente real medida, em vez de confiar só nos defaults de fábrica (3.0 A / 0.5 A).
13. [ ] Meça a corrente do trecho 5V (saída da fonte Hi-Link) com o relé acionado e o Wi-Fi ativo — confirme que fica confortavelmente abaixo de 1 A (ver seção 6).

---

## 8. Referência rápida — BOM

Ver a lista completa com quantidades e observações no [Memorial Descritivo, seção 5](MEMORIAL-DESCRITIVO.md#5-bill-of-materials-bom-definitivo). Resumo do que este manual usa:

- Bomba Intech BSCA2 1/4 HP 220 V
- 2× boia reed switch + resistores 1 kΩ / 4.7 kΩ / 10 kΩ (EOL)
- Disjuntor 6–10 A curva C
- Fonte Hi-Link 5 V/1 A
- ESP32-S3 + YYNMOS-4 (4 canais, só 3 em uso — LED RGB)
- Módulo relé 5 V 1 canal, com optoacoplador (jumper JD-VCC) + Contator monofásico 25 A AC-3 (bobina 220 V)
- PZEM-004T 10 A (shunt interno)
- Capacitor cerâmico 100 nF + resistor pull-up 3.3 kΩ
- Supressor de surto (snubber RC ou varistor 275 V)
- Botão inox antivandalismo com LED RGB anodo comum + cabo UTP Cat5e/6
- 5× microswitch (2+2 nas grades, em série cada par, + 1 porta cozinha) + 2 cabos de 2 vias até as grades

---

## 9. Expansão — Sensores de Porta (Microswitches)

Adiciona 3 entradas digitais ao mesmo quadro, para as 3 portas da área de
serviço: a **grade principal** e a **grade secundária** (2 microswitches
cada, ligados **em série** entre si) e a **porta de madeira da cozinha** (1
microswitch). Cinco switches físicos, mas só 3 sinais chegam ao ESP32 — a
série já resolve "os dois confirmam" no cobre, sem precisar de lógica extra
no YAML. Não mexe na FSM da bomba — são `binary_sensor` independentes, só
para observabilidade no Home Assistant.

### 9.1 Por que a polaridade é o oposto do botão

O botão usa `inverted: true` porque o evento que importa é o **clique**
(pino LOW = "pressionado" = ON). Aqui o que importa é o **repouso**: cada
switch é NA (normally open) e fecha para GND quando **atuado** — lingueta
avançada na grade, ou porta encostada no batente/marco na cozinha.

Sem `inverted`, o `binary_sensor` fica assim:

| Situação física | Pino | Estado ESPHome | Semântica HA (`device_class: lock`/`door`) |
|---|---|---|---|
| Lingueta avançada / porta fechada (switch(es) atuado(s)) | LOW (GND) | `off` | Trancada / Fechada |
| Lingueta recuada / porta aberta (switch solto) | HIGH (pull-up) | `on` | Aberta / Não confirmada |
| **Cabo cortado ou switch desconectado** | HIGH (pull-up) | `on` | **Aberta / Não confirmada** |

A terceira linha é o ponto: um cabo rompido cai no mesmo estado de "não
confirmada" que a porta genuinamente aberta — nunca no de "trancada". É a
mesma filosofia de fail-safe da matriz de boias (seção 4), só que aqui via
polaridade do pull-up em vez de um laço EOL dedicado. Não há resistor de
fim de linha nestes 3 sinais — um corte no meio do cabo também lê como
"aberta", então o efeito prático de segurança é o mesmo sem o custo de mais
resistores e um canal ADC por switch.

> ⚠️ **Nunca use `inverted: true` nestes 3 sinais.** Inverter a leitura faz
> um cabo cortado ler como `off` (Trancada/Fechada) — o disfarce exatamente
> oposto ao que este desenho existe para evitar. `inverted` só entra no
> botão, porque lá o evento é o clique, não o repouso.

**A série faz o "os dois confirmam" na fiação, não no software.** Nas duas
grades, os 2 microswitches ficam em série entre si: o laço só fecha (e o
GPIO só lê LOW/"confirmada") se **ambos** estiverem atuados. Qualquer um
sozinho aberto — ou o cabo rompido em qualquer ponto do trajeto, incluindo
entre os dois switches — já derruba o laço inteiro pra "aberta/não
confirmada". Nenhum `binary_sensor: template` combinando dois pinos é
necessário (a versão anterior deste manual tinha 2 GPIOs + 1 combinada para
uma única grade; esta versão usa 1 GPIO por grade, com a combinação já
feita na série).

### 9.2 Grade Principal e Grade Secundária — cabo novo, 2 vias cada

Em cada grade, os dois microswitches ficam montados na própria fechadura
elétrica, cada um posicionado para ser pressionado pela lingueta quando ela
avança até a posição travada — não pela porta encostando no marco. Isso é o
que garante detectar o caso real que motivou o pedido: a fechadura
"comandada" para travar mas a lingueta não assentou (desalinhamento, sujeira,
o que for). **Se alguma das duas grades não tiver fechadura elétrica** (só
tranca mecânica), o mesmo microswitch serve para confirmar a posição da
grade — só troque o ponto de montagem para o batente, como na cozinha.

Os dois microswitches de uma mesma grade ficam **em série** (perna de saída
do primeiro na perna de entrada do segundo), e só as duas pontas externas
desse par descem ao quadro — **2 vias por grade**, não 3:

| Grade | Via | Função | Vai para (no quadro) |
|---|---|---|---|
| Principal | 1 | GND comum | GND comum |
| Principal | 2 | Sinal (par em série) | GPIO6 (`pino_grade_principal`) |
| Secundária | 1 | GND comum | GND comum |
| Secundária | 2 | Sinal (par em série) | GPIO7 (`pino_grade_secundaria`) |

Cabo por grade: par trançado ou cabo de alarme 2 vias já resolve, **não**
precisa ser blindado. Se o trecho até alguma grade for longo ou correr
perto de fiação de 220 V, considere reforçar o sinal com um pull-up externo
de 10 kΩ (3,3 V) + capacitor cerâmico 100 nF (GND) no lado do quadro — o
pull-up interno do ESP32-S3 (~45 kΩ) é fraco e mais sensível a ruído em
cabos longos. Não é obrigatório para um cabo curto dentro de casa.

> ⚠️ **GPIO8 (Grade Secundária) é pino de strapping** (controla se a ROM
> imprime o log de boot — não decide o modo SPI/Download, que é o GPIO9,
> esse sim evitado de propósito). Efeito esperado é só cosmético, mas
> **ainda não foi testado** com o par em série fechado no instante do boot.
> Antes de fixar a fiação definitiva: feche manualmente os dois microswitches
> da grade secundária, dê power-cycle no ESP32 e confirme no log/Home
> Assistant que ele sobe normalmente (FSM entra em E0 ou E1, não trava em
> bootloop). Se algo der errado, troque `pino_grade_secundaria` para outro
> GPIO livre do seu módulo específico — a troca é só no `substitutions:` do
> YAML.

### 9.3 Porta da cozinha — reaproveita o UTP existente

Nenhum cabo novo: o UTP que já vai do quadro até a cozinha (seção 5) tem 2
vias sobrando (7 e 8). O microswitch fica no batente da porta de madeira,
posicionado para ser pressionado quando a porta fecha. Só 1 switch aqui, sem
série.

| Via (UTP) | Função | Vai para (no quadro) |
|---|---|---|
| 7 | Contato seco — GND | GND comum |
| 8 | Contato seco — sinal | GPIO8 (`pino_porta_cozinha`) |

Confirme com multímetro, no switch isolado (antes de plugar no ESP32), que
a resistência entre as vias 7/8 cai a praticamente zero quando pressionado
e volta a infinita quando solto — mesmo teste já feito para o botão na
seção 5.

### 9.4 Checklist desta expansão

1. [ ] Grade Principal: cabo de 2 vias puxado, sem correr junto de fiação
   de 220 V.
2. [ ] Grade Secundária: idem.
3. [ ] Multímetro, em cada grade, no par em série isolado (antes de plugar
   no ESP32): resistência cai a praticamente zero **só quando os DOIS**
   microswitches estão atuados; qualquer um sozinho solto já mostra
   circuito aberto.
4. [ ] Cada microswitch da grade fecha só quando a lingueta correspondente
   avança até a posição travada (não antes).
5. [ ] Cozinha: continuidade das vias 7/8 do UTP confirmada ponta a ponta.
6. [ ] GPIO8 testado com o par da grade secundária fechado no boot (ver
   9.2) antes de fixar a fiação definitiva.
7. [ ] Após gravar o firmware: `Grade Principal (Serviço)`, `Grade
   Secundária (Serviço)` e `Trava Madeira Cozinha` aparecem no Home
   Assistant e mudam de estado ao atuar cada switch à mão.
