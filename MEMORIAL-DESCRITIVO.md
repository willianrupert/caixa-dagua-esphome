# Memorial Descritivo — Sistema de Bombeamento Cisterna → Caixa D'água

**Plataforma:** ESP32-C3 (conectado via USB) rodando **ESPHome**, integrado ao **Home Assistant**.
**Princípio de projeto:** lógica máxima no firmware, zero solda na eletrônica (bornes/trilho DIN), isolamento industrial e persistência de estado em memória não-volátil.

---

## 0. Credenciais e Rede

| Item | Valor |
|---|---|
| Wi-Fi | 2 redes com fallback automático — ver `secrets.yaml` (**não versionado**) |
| Firmware | ESPHome |
| Integração | Home Assistant (API nativa ESPHome, chave em `secrets.yaml`) |
| Conexão de flash | USB na 1ª gravação; OTA depois |

> Todas as credenciais (SSIDs, senhas, chave da API, senha OTA) vivem em `secrets.yaml`, que está no `.gitignore`. Use o `secrets.example.yaml` como modelo.

---

## 1. Conceitos Fundamentais

### 1.1 Botão físico único (curto vs. longo)
O sistema usa um **timer de intenção** para diferenciar os cliques:

- **Clique Curto (< 1.0 s):** liga/desliga manual da bomba ou reset de falha (atua nos estados E3, E4, E6, E7).
- **Clique Longo (≥ 1.5 s):** interrupção **soberana** — ativa/desativa o **Modo Pausa (E8)** a partir de **qualquer** estado, inclusive falhas críticas (E6/E7).
- A janela entre 1.0 s e 1.5 s é zona morta (evita interpretação ambígua).

### 1.2 Persistência de estado (EEPROM/Flash)
- O ESP32 **não salva o estado do relé físico** — salva o **Estado Lógico da Máquina de Estados**.
- **Gravação por evento:** escreve na flash **somente quando há mudança real de estado** (ex.: Standby → Pré-partida, entrada em Pausa). Nunca gravação periódica.
- Como a bomba liga poucas vezes ao dia, o desgaste da flash é desprezível (durabilidade de décadas).

### 1.3 Boot Recovery (volta da energia)
Se o sistema salvasse "bomba ligada" e religasse o relé direto no boot, haveria quebra de segurança grave (cisterna pode ter secado durante o apagão). Por isso, todo boot passa pela **Rotina de Inicialização Segura (E9)**: relê a matriz de boias e, se for religar o motor, aplica o *blanking time* de 5 s no monitor de corrente antes de validar a partida.

---

## 2. Máquina de Estados Definitiva (FSM)

### Estados base
| Estado | Nome | Descrição |
|---|---|---|
| E0 | Standby | Caixa OK, sistema pronto. LED apagado. |
| E1 | Pré-partida | Delay de 5 s antes de ligar a bomba. |
| E2 | Recalque (Crítico) | Enchendo com nível crítico. LED Roxo/Pink. |
| E3 | Recalque (Intermediário/Manual) | Enchendo em nível intermediário ou por comando manual. LED Azul. |
| E4 | Finalização (Sucesso) | Caixa encheu. LED Verde por 1 minuto. |
| E5 | Falha Contator (Corrente Fantasma) | Corrente detectada com relé desligado = contatos soldados. LED Branco piscando. Requer clique curto (e desligar o disjuntor!). |
| E6 | Falha Corrente Alta | Rotor preso. LED Vermelho piscando. Requer clique curto para reset. |
| E7 | Falha Corrente Baixa | Bomba sem água (cisterna seca). LED Azul piscando. Requer clique curto para reset. |
| E8 | **Modo Pausa** | Manutenção soberana. LED Amarelo **sólido** (manual) ou **piscando** (automática: inching/sensor). |
| E9 | **Boot Seguro** | Rotina de recuperação pós-apagão / pós-pausa. |

### Estado 8 — MODO PAUSA (Manutenção Soberana)
- **Entrada:** botão pressionado por **≥ 1.5 s** em **qualquer** estado (mesmo E6/E7).
- **Ações:**
  - Desliga o motor **imediatamente** (segurança mecânica).
  - Salva "Pausado" na memória não-volátil.
  - LED RGB: **Amarelo sólido indefinido** (Vermelho + Verde, sem Azul).
  - **Ignora completamente** qualquer variação de nível das boias.
- **Saída:** somente novo clique longo (≥ 1.5 s). Ao sair, limpa a memória e transiciona para **E9 (Boot Seguro)** para reavaliar a casa.

### Proteção de Inching — Tempo Máximo de Bomba Ligada
- **Cenário coberto:** a boia superior quebra e nunca sinaliza "caixa cheia" — sem esta regra, a bomba ficaria ligada indefinidamente (transbordo + desgaste do motor).
- **Regra:** a bomba não pode ficar ligada por mais de **10 minutos contínuos** (E2/E3). Ao estourar o limite, o sistema entra em **Pausa Automática (E8)**: desliga tudo, LED **amarelo piscando** (distinto da pausa manual, que é sólido), e fica travado até o clique longo do usuário.
- A contagem segue o **relé físico**, não o estado lógico: a transição E2 → E3 (nível saiu do crítico) **não** reinicia os 10 minutos. Cada nova partida da bomba zera a contagem.
- Por que Pausa e não Falha (E6/E7): estouro de tempo indica problema de **sensor**, não do motor — exige inspeção humana da caixa antes de qualquer religamento, e o E8 é o único estado que ignora completamente as boias.
- Calibrável via substitution `tempo_maximo_bomba` no `caixa-dagua.yaml` (dimensionar folgado acima do tempo real de enchimento).

### Estado 9 — ROTINA DE BOOT SEGURO (Recuperação de Apagão)
- **Entrada:** ESP32 energizado (energia voltou) **ou** saída do Modo Pausa (E8).
- **Ação:** lê o último estado gravado na flash e decide:

| Estado gravado na flash | Transição no boot |
|---|---|
| Modo Pausa (E8) | Permanece **travado em E8** (Amarelo). |
| Standby (E0) ou Finalização (E4) | Vai direto para **E0** (Pronto). |
| Pré-partida (E1) ou Recalque (E2/E3) | Avalia a boia superior. Se a caixa **não** está cheia → **E1 (Pré-partida)**: roda o delay de 5 s, LED pisca as cores correspondentes, liga a bomba e reativa o monitor de corrente do PZEM **do zero** (com blanking de 5 s). |
| Falha (E6/E7) | Retorna ao estado de falha com LED piscando, **exigindo clique curto** para destravar (o usuário fica sabendo que houve falha antes do apagão). |

---

## 3. Fluxo de Cores do LED RGB (com fade in/out suave)

| # | Situação | Cor |
|---|---|---|
| 1 | Caixa OK / Standby | **Apagado** |
| 2 | Caixa vazia (aguardando) | **Vermelho sólido** |
| 3 | Enchendo — nível crítico | **Roxo/Pink** (Vermelho + Azul simultâneos) |
| 4 | Enchendo — nível intermediário / manual | **Azul sólido** |
| 5 | Caixa encheu (sucesso) | **Verde sólido por 1 minuto** |
| 6 | Modo Pausa **manual** (manutenção) | **Amarelo sólido indefinido** (Vermelho + Verde) |
| 7 | Pausa **automática** (inching / falha de sensor) | **Amarelo piscando lento** (700 ms) |
| 8 | Falha corrente alta (rotor preso) | **Vermelho piscando** |
| 9 | Falha corrente baixa (sem água) | **Azul piscando** |
| 10 | Falha de contator (corrente fantasma) | **Branco piscando rápido** (300 ms) — desligue o disjuntor! |

---

## 4. Sensores e Filtros

### 4.1 Matriz de boias em 2 fios — laço SUPERVISIONADO (multiplexação analógica)
- 2 boias reed switch (contato seco), cada uma em série com um resistor:
  - **Nível Mínimo:** resistor de **1 kΩ**
  - **Nível Máximo:** resistor de **4.7 kΩ**
- **Resistor de fim de linha (EOL) de 10 kΩ** em paralelo com as boias, **instalado no lado da caixa** (técnica de central de alarme): é ele que torna o rompimento do cabo eletricamente detectável.
- Os três componentes em **paralelo**, descendo ao quadro por apenas **2 fios**.
- **Pull-up de 3.3 kΩ** do nó do ADC para 3.3 V (referência do divisor, no quadro).
- **Fiação das boias:** contato **fecha quando a água está ABAIXO** da boia. Tensões resultantes:

| Situação | Resistência do laço | Tensão no ADC | Classificação |
|---|---|---|---|
| Cabo **rompido** / boias desconectadas | ∞ | **~3.3 V** | ⚠️ FALHA SENSOR |
| Ambas abertas | 10 kΩ | **~2.48 V** | Caixa **cheia** |
| Só máxima fechada | 4.7k ∥ 10k = 3.2 kΩ | **~1.62 V** | **Intermediário** |
| Ambas fechadas | 1k ∥ 4.7k ∥ 10k = 762 Ω | **~0.62 V** | **Crítico** |
| Cabo em **curto** | ~0 | **~0 V** | ⚠️ FALHA SENSOR |

- Limiares calibráveis: falha >2.90 V; cheia >2.10 V; intermediário >1.10 V; curto <0.35 V.
- **FALHA SENSOR** precisa persistir por 15 leituras (15 s) para disparar — aí o sistema entra em **Pausa Automática (E8, amarelo piscando)** e registra o motivo em "Última Ocorrência". Um pino flutuando (nada conectado, caso da bancada) também cai aqui.
- **Capacitor cerâmico 100 nF** entre o pino ADC e GND para filtrar ruído do cabo.
- **Histerese anti-marola:** mudança de nível só é aceita após **4 leituras consecutivas** (4 s) — evita que a ondulação da água na boia cause oscilação E2↔E3, piscadas no LED e gravações desnecessárias na flash.

### 4.2 Monitoramento de corrente (PZEM-004T)
- PZEM-004T (versão 10A, shunt interno) na fase de 220 V da bomba; dados via serial (UART) para o ESP32.
- **Blanking time de 5 s** na partida (cegueira de arranque) antes de validar corrente.
- **Corrente alta** → rotor preso → **E6**.
- **Corrente baixa** → bomba a seco / cisterna vazia → **E7**.
- **Corrente fantasma** (relé desligado + corrente > 0.5 A por mais de 10 s) → contatos do relé/contator **soldados** → **E5** (branco piscando). O software não consegue cortar um defeito mecânico — o alarme existe para o usuário **desligar o disjuntor**. Vale em **qualquer** estado, inclusive durante a Pausa (é exatamente quando alguém pode estar com a mão na caixa!).

---

## 5. Bill of Materials (BOM) Definitivo

### 5.1 Cisterna (Hidráulica e Mecânica)
| Qtd | Item | Observações |
|---|---|---|
| 1 | Bomba Submersa Intech BSCA2 (1/4 HP) | Motor 220 V |
| 1 | Tubo PVC Esgoto 100 mm (branco), 80–85 cm | Cobre a bomba inteira (da base do motor até passar do bocal de recalque) |
| 1 | CAP (tampão) de esgoto 100 mm | Furo central no diâmetro exato do cano de 1" do recalque; + 1 furo minúsculo (2–3 mm) na face superior para alívio da bolha de ar |
| 1 | Mangueira PEAD 1" (faixa azul) | Conectada à bomba com conector de compressão mecânico (latão ou plástico) |
| — | Cordas de nylon náutica 8 mm + parabolts | Fixação e suspensão da bomba com **inclinação de 15°** |

### 5.2 Caixa D'água Superior (Sensores de Nível)
| Qtd | Item | Observações |
|---|---|---|
| 2 | Boias de nível reed switch | Contato seco, magnéticas |
| 1 | Resistor 1 kΩ | Em série com a boia de nível **Mínimo** |
| 1 | Resistor 4.7 kΩ | Em série com a boia de nível **Máximo** |
| 1 | Resistor 10 kΩ (fim de linha / EOL) | Em **paralelo** com as boias, no lado da caixa — supervisão do cabo (rompimento/curto detectáveis) |
| — | Fiação | Conjuntos em paralelo → **apenas 2 fios** descem ao quadro |

### 5.3 Quadro de Comando (Eletrônica e Potência) — Zero Solda
**Alimentação e Proteção:**
| Qtd | Item | Observações |
|---|---|---|
| 1 | Disjuntor termomagnético monopolar 6A ou 10A, curva C | Dedicado ao circuito 220 V da bomba |
| 1 | Fonte chaveada trilho DIN 5V / 2A | Alimenta ESP32, LEDs da cozinha e módulos |

**Processamento e Sinal:**
| Qtd | Item | Observações |
|---|---|---|
| 1 | **ESP32-C3** (via USB, firmware ESPHome) | Cérebro local: FSM, leitura das boias, PWM dos LEDs |
| 1 | Placa interface MOSFET **YYNMOS-4** (4 canais, 3 em uso) | Recebe 3.3 V do ESP32, libera 5 V nos bornes. Canais 1–3: LED RGB da cozinha; canal 4 não é usado |

**Potência e Acionamento (cascata):**
| Qtd | Item | Observações |
|---|---|---|
| 1 | Módulo relé 5 V 1 canal, com optoacoplador (jumper JD-VCC) | Recebe sinal direto do GPIO7 do ESP32 (IN ativo em LOW); fecha o circuito AC da bobina do contator |
| 1 | Contator monofásico 12A, categoria AC-3, bobina 220 V | Acionado pelo relé; é quem liga a bomba de fato |

**Telemetria e Filtros Passivos:**
| Qtd | Item | Observações |
|---|---|---|
| 1 | PZEM-004T (versão 10A, shunt interno) | Fase da bomba passa pelos parafusos; serial → ESP32 |
| 1 | Capacitor cerâmico 100 nF | Entre pino ADC e GND (filtro de ruído das boias) |
| 1 | Resistor 3.3 kΩ (pull-up do ADC) | Do nó do ADC para 3.3 V — referência do divisor da matriz de boias |
| 1 | Supressor de surto (Snubber RC ou Varistor 275 V) | Nos bornes A1/A2 da bobina do contator (arco de desligamento) |

### 5.4 Cozinha (Interface Humana)
| Qtd | Item | Observações |
|---|---|---|
| 1 | Botão pulsador antivandalismo inox 19 ou 22 mm | Momentâneo (sem trava), LED RGB integrado 3–6 V, **anodo comum** |
| 1 | Cabo de rede UTP Cat5e/Cat6 | Único cabo quadro → cozinha, usando 6 das 8 vias: 1× 5 V constante (anodo), 3× R/G/B (canais 1–3 do YYNMOS-4), 2× contato seco do botão (GND + pino digital) |
| 1 | Espelho cego premium (opcional) | Se instalado em caixa 4x2 de parede em vez de embutido na marcenaria |

---

## 6. Regras Estritas de Segurança (resumo)

1. **Nunca religar a bomba direto no boot** — todo boot passa pela E9 (Boot Seguro) com reteste de boias e blanking de 5 s no PZEM.
2. **Pausa é soberana** — clique longo desliga o motor imediatamente de qualquer estado e trava o sistema até novo clique longo.
3. **Falhas exigem reconhecimento humano** — E6/E7 só destravam com clique curto, inclusive após apagão.
4. **Flash só grava em mudança de estado** — jamais gravação periódica (proteção contra desgaste).
5. **Modo Pausa ignora boias** — nenhuma variação de nível liga a bomba durante manutenção.
6. **Inching de 10 minutos** — a bomba nunca fica ligada mais de 10 min contínuos; se estourar (boia superior quebrada?), entra em Pausa Automática (amarelo piscando) e exige clique longo para destravar.
7. **Nunca reiniciar por falta de rede** — `reboot_timeout: 0s` no Wi-Fi e na API. Sem isso, o padrão do ESPHome reiniciaria o chip a cada 15 min sem Wi-Fi/HA, violando a autonomia local.
8. **Corrente fantasma alarma sempre** — corrente com relé desligado = contator soldado (E5, branco piscando), detectado em qualquer estado, inclusive na Pausa.
9. **Sensor de nível supervisionado** — cabo rompido, em curto ou desconectado é detectado pelo laço EOL e leva à Pausa Automática; o sistema nunca opera "no escuro".

---

## 7. Notas para implementação em ESPHome

- **FSM:** implementada em `caixa-dagua.yaml` com `globals` (`restore_value: true`) + `preferences: flash_write_interval: 0s` — grava na NVS imediatamente, mas só quando o estado muda (regra da seção 1.2).
- **Botão:** `binary_sensor` (GPIO com pull-up) usando `on_multi_click` para separar clique curto (< 1.0 s) de clique longo (≥ 1.5 s, dispara ainda pressionado).
- **LED RGB:** 3 saídas `ledc` (PWM) → `light: rgb` com `transition` para os fades suaves. O botão é anodo comum, **mas** o YYNMOS-4 é chave low-side: sinal ALTO do ESP32 liga o canal e acende o LED → **não usar `inverted`** (só inverteria se o LED fosse ligado direto no GPIO).
- **Boias:** `adc` sensor no pino analógico com filtro `median`, histerese de 4 leituras e supervisão do laço EOL (faixas válidas + zonas de falha).
- **PZEM-004T:** componente nativo `pzemac` (V3/Modbus) via UART.
- **Relé:** `switch: gpio` direto no GPIO7 → módulo relé 5V (IN ativo em LOW, `pin: { inverted: true }` no ESPHome), com `restore_mode: ALWAYS_OFF` (a E9 decide religar, nunca o boot cru). O pull-up de IN do próprio módulo mantém o relé desligado enquanto o GPIO flutua no boot, antes do ESPHome assumir o pino — camada extra de segurança sobre o `ALWAYS_OFF`.
- **Autonomia:** `reboot_timeout: 0s` em `wifi:` e `api:` — o chip jamais reinicia por falta de rede.
- **Home Assistant:** expor estado da FSM (`text_sensor`), nível, "Última Ocorrência" e telemetria do PZEM; dois `button` template espelham os cliques físicos. Métricas de debug ficam **ocultas** no registro de entidades do HA (não no YAML). A lógica de segurança roda **100% local no ESP32**, independente do Wi-Fi/HA.

As credenciais ficam em `secrets.yaml` (fora do Git) — copie o `secrets.example.yaml` e preencha.
