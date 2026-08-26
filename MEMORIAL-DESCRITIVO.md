# Memorial Descritivo — Sistema de Bombeamento Cisterna → Caixa D'água

**Plataforma:** ESP32-S3 (conectado via USB) rodando **ESPHome**, integrado ao **Home Assistant**.
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

- **Clique Curto (< 1.0 s):** liga/desliga manual da bomba ou reset de falha (atua nos estados E0, E3, E6, E7).
- **Clique Longo (≥ 1.5 s):** interrupção **soberana** — ativa/desativa o **Modo Pausa (E8)** a partir de **qualquer** estado, inclusive falhas críticas (E6/E7).
- A janela entre 1.0 s e 1.5 s é zona morta (evita interpretação ambígua).
- **Clique com a caixa já cheia não muda de estado — só pisca.** Em E0
  (parado, caixa cheia) o clique dá um pulso de verde e volta a apagar; em E4
  (verde já aceso, dentro da janela de 1 min) o clique pisca e retoma o verde
  sólido. Nenhum dos dois casos toca em `fsm_state`, grava na flash ou
  reinicia o `sucesso_timer` — existe só pra confirmar que o clique chegou,
  sem deixar clique repetido monopolizar o LED com rodadas de 60 s.

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
| E5 | Falha Contator (Corrente Fantasma) | Corrente detectada com relé desligado = contatos soldados **ou chave de bypass em MANUAL** (ver 4.2.1). LED Branco piscando. Requer clique curto (e desligar o disjuntor, se não for o bypass). |
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
| 10 | Falha de contator (corrente fantasma) | **Branco piscando rápido** (300 ms) — desligue o disjuntor, salvo se a chave estiver em MANUAL |

### 3.1 Varredura de matiz no boot

Ao energizar, o botão varre 360° de matiz (~1,9 s) e se resolve na cor do
estado que a rotina E9 decidir. Serve para duas coisas além do agrado:
prova, num relance, que os três canais do LED estão vivos e na cor certa
— um canal queimado ou um fio de catodo trocado aparece na hora, sem
precisar forçar estado nenhum — e dá sinal visível de que a energia voltou.

**Não custa tempo de boot.** Ela roda *dentro* dos 2 s que a E9 já gastava
parada estabilizando a leitura do ADC (`boot_seguro` dispara o script e
segue para o `delay`, sem bloquear). O tempo era o mesmo; antes não tinha
o que mostrar.

**Decoração nunca ganha do estado.** `goto_state` interrompe a animação
(`script.stop`) antes de pintar qualquer cor. Sem isso, uma partida
automática disparada pela primeira leitura do ADC — que acontece a ~1 s,
antes da E9 concluir — ficaria escondida atrás do arco-íris ainda em curso.

Implementação: 24 passos de 15°, com transição (75 ms) deliberadamente
maior que o intervalo entre passos (70 ms), para que cada comando
interrompa o fade anterior no meio. A cor nunca "chega e senta" — é
movimento contínuo, não 24 saltos.

---

## 4. Sensores e Filtros

### 4.1 Matriz de boias em 2 fios — laço SUPERVISIONADO (multiplexação analógica)
- 2 boias reed switch (contato seco), cada uma em série com um resistor:
  - **Nível Mínimo:** resistor de **1 kΩ**
  - **Nível Máximo:** resistor de **4.7 kΩ**
- **Resistor de fim de linha (EOL) de 10 kΩ** em paralelo com as boias, **instalado no lado da caixa** (técnica de central de alarme): é ele que torna o rompimento do cabo eletricamente detectável.
- Os três componentes em **paralelo**, descendo ao quadro por apenas **2 fios**.
- **Pull-up de 3.3 kΩ** do nó do ADC para 3.3 V (referência do divisor, no quadro).
- **Fiação das boias:** contato **fecha quando a água está ABAIXO** da boia — as duas. Isso é requisito de fail-safe, não convenção arbitrária: ver 4.1.1.

| Situação | Resistência do laço | Tensão no ADC | Classificação |
|---|---|---|---|
| Cabo **rompido** / boias desconectadas | ∞ | **~3.3 V** | ⚠️ FALHA SENSOR |
| Ambas abertas | 10 kΩ | **~2.48 V** | Caixa **cheia** |
| Só máxima fechada | 4.7k ∥ 10k = 3.2 kΩ | **~1.62 V** | **Intermediário** |
| Ambas fechadas | 1k ∥ 4.7k ∥ 10k = 762 Ω | **~0.62 V** | **Crítico** |
| Cabo em **curto** | ~0 | **~0 V** | ⚠️ FALHA SENSOR |

- Limiares calibráveis: falha >2.90 V; cheia >2.10 V; intermediário >1.10 V; curto <0.35 V.

#### 4.1.1 Por que as duas boias fecham secas — e por que não se gira

Em 2026-08-25 as duas boias foram giradas 180° para tratar um transbordo, e a
reavaliação que isso forçou mostrou que girar quebra **duas** coisas
independentes. A causa real do transbordo era uma infiltração na entrada de
água da caixa (vazamento pequeno e contínuo), corrigida na hidráulica — que é
onde problema hidráulico se resolve. As boias voltaram à orientação original.
Fica registrado para que ninguém refaça o caminho:

**1. "Cheia" precisa ser a ausência de sinal, não a presença.** Com as boias
fechando secas, caixa cheia = as duas abertas = só o EOL no laço. Um reed
morto, um fio rompido ou uma emenda corroída produzem exatamente a mesma
leitura de "aberto" — ou seja, **defeito no ramo da máxima faz o sistema ler
"cheia" e desligar a bomba**. Invertendo a máxima, "cheia" passaria a exigir
um contato vivo e fechado, e o mesmo defeito viraria "ainda não encheu":
bomba segue ligada, transbordo, com o inching de 10 min como única rede. É a
diferença entre falhar para o lado seguro e falhar para o lado que molha a
casa. (O EOL **não** cobre esse caso: ele detecta rompimento no trecho de 2
fios compartilhado; um rompimento só no ramo da máxima deixa o laço vivo
pelos outros ramos.)

**2. Com as duas fechando molhadas, a matemática do divisor colapsa.**
"Intermediário" viraria 1k ∥ 10k = 909 Ω e "cheia" viraria 1k ∥ 4.7k ∥ 10k =
762 Ω — **~90 mV de separação entre dois estados normais**, que se alternam a
cada enchimento, indistinguíveis dentro da tolerância de ±5% dos resistores.
E a distinção perdida é a mais crítica do sistema: cheia desliga a bomba,
intermediário continua enchendo. Na orientação correta a menor folga entre
estados vizinhos é ~0,86 V, e ~120 Ω de emenda oxidada (que na configuração
colapsada já bastavam para confundir os dois) passam a mover a leitura
poucos milivolts.

**O que a orientação correta troca em retorno:** um contato **colado fechado**
na máxima passa a ser o modo inseguro (lê "intermediário" com a caixa cheia →
bomba segue). É uma troca deliberada e favorável — reed de baixa corrente
soldando é bem menos provável que fio rompido ou contato corroído, e o
inching cobre o caso, com a mensagem que ele já grava apontando para o lugar
certo (*"N min sem encher - verificar boia superior"*).

**Independente da orientação:** boia travada mecanicamente na posição baixa
(lodo, braço emperrado) significa "ainda não encheu" nos dois desenhos, e
boia mínima morta nunca produz "crítico" — logo a auto-partida, que exige
`lvl == 2`, nunca dispara e a caixa seca em silêncio. Girar as boias não
resolvia nenhum dos dois.
- **FALHA SENSOR** precisa persistir por 15 leituras (15 s) para disparar — aí o sistema entra em **Pausa Automática (E8, amarelo piscando)** e registra o motivo em "Última Ocorrência". Um pino flutuando (nada conectado, caso da bancada) também cai aqui.
- **Capacitor cerâmico 100 nF** entre o pino ADC e GND para filtrar ruído do cabo.
- **Histerese anti-marola:** mudança de nível só é aceita após **4 leituras consecutivas** (4 s) — evita que a ondulação da água na boia cause oscilação E2↔E3, piscadas no LED e gravações desnecessárias na flash.

**Pendência — o "estado fantasma" (boia mínima presa) não é distinguível do
crítico com os resistores atuais.** Existe uma 6ª combinação, fisicamente
impossível em operação normal mas alcançável por defeito mecânico: mínima
travada fechada (lodo/sujeira) enquanto a máxima lê aberta de verdade —
1k ∥ 10k ≈ 909 Ω, **~0.71 V**. Avaliado e **não implementado**: 909 Ω fica
próximo demais dos 762 Ω do crítico normal (~90 mV) para separar com
segurança usando resistores de tolerância padrão (±5%) — uma janela de
detecção ali tem risco real de confundir as duas coisas nos dois sentidos:
ler "crítico" de verdade como falha (trava a bomba à toa) ou ler o defeito
mecânico como "crítico" normal (liga a bomba com a mínima mentindo). Hoje
esse 6º caso cai silenciosamente em **Crítico (E2)**, que liga a bomba.

É consequência de escolher 1k/4.7k/10k, que separa bem as 3 combinações do
caminho normal mas não a 4ª. Solução de verdade exigiria resistores
escolhidos para separar as **4** combinações possíveis — mudança de
projeto, não de software, e a caixinha já está potada. Registrar para a
próxima geração de sensores de nível.

### 4.2 Monitoramento de corrente (PZEM-004T)
- PZEM-004T (versão 10A, shunt interno) na fase de 220 V da bomba; dados via serial (UART) para o ESP32.
- **Blanking time de 5 s** na partida (cegueira de arranque) antes de validar corrente.
- **Corrente alta** → rotor preso → **E6**.
- **Corrente baixa** → bomba a seco / cisterna vazia → **E7**.
- **Corrente fantasma** (relé desligado + corrente > 0.5 A por mais de 10 s) → contatos do relé/contator **soldados** → **E5** (branco piscando). O software não consegue cortar um defeito mecânico — o alarme existe para o usuário **desligar o disjuntor**. Vale em **qualquer** estado, inclusive durante a Pausa (é exatamente quando alguém pode estar com a mão na caixa!).

#### 4.2.1 A chave de bypass produz a mesma assinatura — e alarma junto

Existe uma chave SPDT que comuta quem alimenta a bobina do contator: o relé
(AUTO) ou a fase direta (MANUAL). Ela existe para **um caso só — o ESP32
quebrou e a casa precisa de água** — e o que faz essa garantia valer é o
caminho em MANUAL não tocar em nada eletrônico: `disjuntor → chave → A1`,
sem passar pelo relé, pelo ESP32 ou pela fonte. Com os três queimados, a
bomba ainda liga. Fiação em [Manual de Fiação, seção 10](MANUAL-FIACAO.md#10-chave-de-bypass--ligar-a-bomba-com-o-esp32-morto).

Consequência: em MANUAL o motor gira sem o relé ter mandado, que é
**exatamente** a assinatura de contator soldado. Não há como distinguir
pela corrente — as duas situações são eletricamente idênticas para o PZEM.

**O alarme continua disparando, de propósito.** Suprimi-lo exigiria um
sinal dizendo ao firmware que o bypass está ativo, e a chave de 3 pinos não
tem contato sobrando para isso. A alternativa — um interruptor no Home
Assistant que o operador liga junto — foi descartada: um esquecimento
deixaria a única detecção de contator soldado desligada por tempo
indeterminado, trocando um incômodo por um risco real.

O que mudou foi só o **texto**, que antes mentia por omissão: a ocorrência
agora diz *"contator soldado OU chave em MANUAL"* em vez de acusar defeito
quando pode ser o operador.

**O teste do bypass se faz com o ESP32 desligado.** É o cenário para o qual
ele existe, portanto o único teste fiel — e de quebra não gera alarme
nenhum, porque não há firmware rodando. Testar com o sistema vivo dispara
E5 em segundos, e o clique curto não adianta enquanto a corrente persistir:
ele limpa, e o alarme volta.

### 4.3 Sensores de porta (microswitches) — fail-safe por polaridade e por série, não por laço EOL

Expansão sem relação com a FSM da bomba: as 3 portas da área de serviço
(grade principal, grade secundária e porta de madeira da cozinha), todas
como `binary_sensor` de observabilidade no Home Assistant. Nas duas grades,
2 microswitches por grade (confirmam que a lingueta da fechadura avançou
até a posição travada, não só que a porta encostou) ficam ligados **em
série entre si** — 5 switches físicos, mas só 3 sinais chegam ao ESP32, um
por porta.

Mesma convenção elétrica do botão (contato NA, pull-up interno do
ESP32-S3), mas com a polaridade **oposta**: sem `inverted`. O botão precisa
que "pressionado" vire `on` (é o evento); aqui o que importa é o **estado
de repouso** — pino em pull-up (HIGH, sem nada puxando pra GND) já
representa "aberta/não confirmada", que é exatamente o estado de um cabo
cortado ou de um switch desconectado. Resultado: falha de fiação e porta
genuinamente aberta caem no mesmo estado no Home Assistant, nunca no de
"trancada" — o mesmo princípio de fail-safe da matriz de boias (4.1), só
que resolvido pela polaridade do pull-up em vez de um laço EOL dedicado com
resistor de fim de linha. Mais barato (nenhum resistor extra), com a
contrapartida de não distinguir "aberta de verdade" de "cabo cortado" — um
laço EOL faria essa distinção, mas para um sensor de porta essa diferença
importa menos do que para o nível d'água (lá a FSM decide ligar a bomba
sozinha; aqui é só alerta pro morador olhar).

**A série resolve "os dois confirmam" na fiação, não no software.** Uma
versão anterior deste sensor usava 2 GPIOs por grade e um
`binary_sensor: template` combinando os dois (`A || B`) num terceiro
sensor "confirmada". Ligar os dois microswitches em série antes de descer
o fio faz a mesma coisa com um GPIO a menos e nenhuma lógica extra: o laço
só fecha se ambos estiverem atuados, e qualquer um sozinho aberto — ou o
cabo rompido em qualquer ponto, inclusive entre os dois switches — já
derruba o laço inteiro. A única perda é a granularidade (não dá pra saber,
de longe, qual dos dois microswitches falhou) — irrelevante aqui, já que o
morador vai até a grade olhar de qualquer forma.

Detalhamento de fiação, valores de pino e o checklist de instalação estão
no [Manual de Fiação, seção 9](MANUAL-FIACAO.md#9-expansão--sensores-de-porta-microswitches).

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
| 1 | Fonte Hi-Link 5V / 1A (módulo AC-DC) | Alimenta ESP32, LEDs da cozinha e módulos — ver Manual de Fiação seção 6 para o orçamento de corrente |

**Processamento e Sinal:**
| Qtd | Item | Observações |
|---|---|---|
| 1 | **ESP32-S3** (via USB, firmware ESPHome) | Cérebro local: FSM, leitura das boias, PWM dos LEDs |
| 1 | Placa interface MOSFET **YYNMOS-4** (4 canais, 3 em uso) | Recebe 3.3 V do ESP32, libera 5 V nos bornes. Canais 1–3: LED RGB da cozinha; canal 4 não é usado |

**Potência e Acionamento (cascata):**
| Qtd | Item | Observações |
|---|---|---|
| 1 | Módulo relé 5 V 1 canal, com optoacoplador (jumper JD-VCC) | Recebe sinal direto do GPIO21 do ESP32 (IN ativo em HIGH); fecha o circuito AC da bobina do contator |
| 1 | Contator monofásico 25A, categoria AC-3, bobina 220 V | Acionado pelo relé; é quem liga a bomba de fato |

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
| 1 | Cabo de rede UTP Cat5e/Cat6 | Único cabo quadro → cozinha, usando as 8 vias: 1× 5 V constante (anodo), 3× R/G/B (canais 1–3 do YYNMOS-4), 2× contato seco do botão (GND + pino digital), 2× contato seco da porta da cozinha (GND + pino digital, ver 5.5) |
| 1 | Espelho cego premium (opcional) | Se instalado em caixa 4x2 de parede em vez de embutido na marcenaria |

### 5.5 Expansão — Sensores de Porta (Microswitches)
| Qtd | Item | Observações |
|---|---|---|
| 2 | Microswitch (contato NA) | Grade Principal — em série entre si, atuados pela lingueta ao avançar (não pela porta encostando) |
| 2 | Microswitch (contato NA) | Grade Secundária — idem, par em série independente da Grade Principal |
| 1 | Microswitch (contato NA) | Porta de madeira da cozinha — montado no batente, atuado pelo fechamento da porta |
| — | Cabo 2 vias (par trançado ou alarme) × 2 | Quadro → cada grade, curto (grades ficam perto do quadro) — 2 vias por grade, não 3: a série já fica do lado de fora |
| — | Fiação da porta da cozinha | Reaproveita as 2 vias do UTP da cozinha até então não usadas (7 e 8) — nenhum cabo novo |
| 2 (opcional) | Resistor 10 kΩ (pull-up externo) + Capacitor cerâmico 100 nF | Um por grade, só se o trecho for longo/ruidoso — o pull-up interno do ESP32-S3 basta para cabo curto. Ver Manual de Fiação 9.2 |

---

## 6. Regras Estritas de Segurança (resumo)

1. **Nunca religar a bomba direto no boot** — todo boot passa pela E9 (Boot Seguro) com reteste de boias e blanking de 5 s no PZEM.
2. **Pausa é soberana** — clique longo desliga o motor imediatamente de qualquer estado e trava o sistema até novo clique longo.
3. **Falhas exigem reconhecimento humano** — E6/E7 só destravam com clique curto, inclusive após apagão.
4. **Flash só grava em mudança de estado** — jamais gravação periódica (proteção contra desgaste).
5. **Modo Pausa ignora boias** — nenhuma variação de nível liga a bomba durante manutenção.
6. **Inching de 10 minutos** — a bomba nunca fica ligada mais de 10 min contínuos; se estourar (boia superior quebrada?), entra em Pausa Automática (amarelo piscando) e exige clique longo para destravar.
7. **Nunca reiniciar por falta de rede** — `reboot_timeout: 0s` no Wi-Fi e na API. Sem isso, o padrão do ESPHome reiniciaria o chip a cada 15 min sem Wi-Fi/HA, violando a autonomia local.
8. **Corrente fantasma alarma sempre** — corrente com relé desligado = contator soldado (E5, branco piscando), detectado em qualquer estado, inclusive na Pausa. A chave de bypass em MANUAL produz a mesma assinatura e também alarma: é deliberado, ver 4.2.1.
9. **Sensor de nível supervisionado** — cabo rompido, em curto ou desconectado é detectado pelo laço EOL e leva à Pausa Automática; o sistema nunca opera "no escuro".

---

## 7. Notas para implementação em ESPHome

- **FSM:** implementada em `caixa-dagua.yaml` com `globals` (`restore_value: true`) + `preferences: flash_write_interval: 0s` — grava na NVS imediatamente, mas só quando o estado muda (regra da seção 1.2).
- **Botão:** `binary_sensor` (GPIO com pull-up) usando `on_multi_click` para separar clique curto (< 1.0 s) de clique longo (≥ 1.5 s, dispara ainda pressionado).
- **LED RGB:** 3 saídas `ledc` (PWM) → `light: rgb` com `transition` para os fades suaves. O botão é anodo comum, **mas** o YYNMOS-4 é chave low-side: sinal ALTO do ESP32 liga o canal e acende o LED → **não usar `inverted`** (só inverteria se o LED fosse ligado direto no GPIO).
- **Boias:** `adc` sensor no pino analógico com filtro `median`, histerese de 4 leituras e supervisão do laço EOL (faixas válidas + zonas de falha).
- **PZEM-004T:** componente nativo `pzemac` (V3/Modbus) via UART.
- **Sensores de porta (expansão):** 3 `binary_sensor: gpio` independentes (Grade Principal, Grade Secundária, Trava Madeira Cozinha), pull-up interno, contato NA, **sem** `inverted` — a polaridade oposta ao botão é deliberada, ver 4.3. Nas grades, os 2 microswitches ficam em série na própria fiação (não há `binary_sensor: template` combinando pinos — a série já faz o "os dois confirmam"). Não participam da FSM da bomba — são só observabilidade.
- **Relé:** `switch: gpio` direto no GPIO21 → módulo relé 5V (IN ativo em HIGH, sem `inverted` no ESPHome — GPIO21 em nível ALTO já liga o relé), com `restore_mode: ALWAYS_OFF` (a E9 decide religar, nunca o boot cru). Confirme no datasheet do módulo o estado do IN com o GPIO flutuando no boot, antes do ESPHome assumir o pino — idealmente o relé fica desligado nesse instante, reforçando o `ALWAYS_OFF`.
- **Autonomia:** `reboot_timeout: 0s` em `wifi:` e `api:` — o chip jamais reinicia por falta de rede.
- **Home Assistant:** expor estado da FSM (`text_sensor`), nível, "Última Ocorrência" e telemetria do PZEM; dois `button` template espelham os cliques físicos. Métricas de debug ficam **ocultas** no registro de entidades do HA (não no YAML). A lógica de segurança roda **100% local no ESP32**, independente do Wi-Fi/HA.

As credenciais ficam em `secrets.yaml` (fora do Git) — copie o `secrets.example.yaml` e preencha.
