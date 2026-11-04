# Cockpit de Telemetria GPS, Performance & Fusão de Datalog

Implementação cirúrgica de uma tela dedicada de telemetria via satélite e análise dinâmica veicular (acessível como nova aba no Modo Mobile e workspace dedicado no Desktop), combinando velocímetro GPS de alta visibilidade, radar vetorial offline do percurso colorido por telemetria do motor, cronômetro de arrancada, cálculo de marcha, aferição de sensor VSS e gravação automática nos arquivos de log `.msl` sempre que o GPS estiver ativo.

## Revisão do Usuário & Decisões Confirmadas

> [!IMPORTANT]
> Todas as preferências funcionais abaixo foram confirmadas na etapa de alinhamento e orientam uma integração modular isolada, preservando integralmente a comunicação serial de 139 bytes da ECU, os mapas existentes e o fluxo atual de Datalog quando o GPS estiver desligado.

- **Acesso Multiplataforma**: Nova aba dedicada na barra inferior do Modo Mobile somada a uma aba/atalho correspondente no Modo Desktop para inspeção e testes em tela ampla.
- **Visualização Offline do Trajeto**: Radar vetorial em Canvas 2D operando 100% offline (ideal para autódromos, pistas de arrancada e rodovias sem sinal celular), com traçado colorido dinamicamente por Velocidade, AFR, Pressão MAP ou RPM.
- **Pacote Completo de Instrumentos de Pista & Rua**:
  - Cronômetro automático de performance (`0–60 km/h`, `0–100 km/h`, `100–200 km/h`, `60 pés`, `201m` e `402m` com velocidade de passagem).
  - Indicador de marcha atual por relação `km/h por 1000 RPM` com aprendizado rápido por marcha e painel comparativo `GPS vs VSS` com sugestão automática de correção de pulsos por km.
  - Odômetro parcial (*Trip*) com filtro anti-oscilação parado, velocidade máxima da sessão, altitude, bússola vetorial, indicador de precisão em metros e trava de tela acesa (*Wake Lock*).
  - Simulador de rota GPS integrado (Arrancada 402m, Circuito Misto e Cruzeiro de Aferição VSS) para validação completa em bancada sem precisar deslocar o veículo.
- **Gravação Inteligente em Datalog (`.msl`)**: Inclusão automática das colunas padronizadas para MegaLogViewer (`GPS Speed`, `Latitude`, `Longitude`, `Altitude`, `Heading`, `GPS Accuracy`, `GPS Gear` e `GPS Trip`) exclusivamente quando o receptor GPS real ou o simulador estiver ativo durante a captura.

---

## 1. Visão Geral & Conceito Central

- **O que entrega**: Um computador de bordo e telemetria de pista integrado ao ecossistema Speeduino. Ele transforma o GPS do smartphone (ou o simulador de bancada) em uma fonte sincronizada de velocidade real, posição geográfica, parciais de arrancada e diagnóstico de transmissão, cruzando esses dados em tempo real com `RPM`, `AFR`, `MAP`, `TPS` e `VSS` vindos da ECU.
- **Público e Contexto de Uso**: Preparadores, pilotos de track day/arrancada e entusiastas que utilizam o celular no suporte de painel conectado via USB OTG ou Bluetooth — especialmente em veículos sem sensor de velocidade de roda configurado na Speeduino ou que precisam calibrar o `VSS` e analisar o comportamento de mistura/ponto em cada curva do traçado no MegaLogViewer.
- **Valor Principal**: Elimina a necessidade de equipamentos externos de medição para aferir velocímetro, descobrir relações de marcha, cronometrar puxadas e registrar coordenadas geográficas sincronizadas quadro a quadro com os sensores vitais do motor.

---

## 2. Experiência do Usuário & Design Visual

- **Fluxos Principais do Usuário**:
  1. **Ativação Rápida e Trava de Tela**: Ao abrir a aba `GPS` no Mobile (ou a visão `GPS & Track` no Desktop), o usuário encontra um cabeçalho enxuto com o seletor **Iniciar GPS** (alta precisão), o botão **Simular Rota** (para testes em bancada), o interruptor **Manter Tela Acesa** (*Wake Lock*) e o controle sincronizado de **Gravar Log** da aplicação.
  2. **Leitura de Condução e Cruzamento com a ECU**: O quadrante principal destaca a **Velocidade GPS (`km/h`)** em tipografia numérica tabular de alto contraste ao lado da **Marcha Estimada (`N` / `1`–`6`)**, bússola cardinal, altitude (`m`), precisão (`± m`) e odômetro parcial (`Trip km`) com botão de zerar e registro de `Vmax`.
  3. **Radar Vetorial Offline do Percurso**: Logo abaixo (no Mobile) ou ao centro-direita (no Desktop), o Canvas 2D desenha em tempo real a trilha percorrida com escala automática em metros, marcador de largada, pino de velocidade máxima, seta direcional do veículo e seletor rápido de cor da trilha (**Velocidade**, **AFR**, **MAP** ou **RPM**), além de alternância entre orientação **Norte Acima** e **Frente Acima** e gestos de zoom/arraste.
  4. **Cronômetro de Arrancada (`Drag Timer`)**: Acionável em sub-aba ou painel dedicado, arma automaticamente quando o veículo para (`< 2 km/h`) e dispara assim que detecta movimento consistente, registrando parciais interpoladas (`0–60`, `0–100`, `100–200 km/h`, `60 ft`, `201m` e `402m` + *Trap Speed*) e guardando o histórico das puxadas da sessão.
  5. **Calibração de Marchas & Aferição `GPS vs VSS`**: Exibe lado a lado a velocidade real por satélite e a velocidade lida pela ECU (`VSS`), o erro percentual e o valor exato sugerido de `Pulses/km` para correção na calibração da Speeduino, além de botões de toque único para gravar a relação `km/h por 1000 RPM` de cada marcha enquanto o motorista trafega estabilizado.

- **Identidade Visual & Tema**:
  - *Direção Estética*: Instrumentação automotiva de competição noturna/diurna, alinhada nativamente à identidade escura já existente no aplicativo, sem poluição visual ou ornamentos artificiais.
  - *Regra de Cores 60-30-10*:
    - **60% Base Dominante**: Ardósia profunda (`slate-950` / `slate-900`) garantindo conforto noturno na cabine e alto contraste sob luz solar.
    - **30% Superfícies Estruturais**: Painéis planos `slate-900/90` com divisórias finas de `1px` (`border-slate-800`) e textos secundários neutros (`slate-400`).
    - **10% Acentos Funcionais de Telemetria**: Ciano (`cyan-400`) para foco ativo e velocidade primária, Esmeralda (`emerald-400`) para fixação de satélite válida e parciais concluídas, Âmbar (`amber-400`) para sinal de baixa precisão/aquisição, e Rosa (`rose-400`) exclusivamente para gravação de Datalog ativa ou alertas reais de mistura/erro.
  - *Tipografia & Disciplina Numérica*: Todos os dígitos de velocidade, coordenadas, cronômetro, odômetro e relações utilizam estritamente numerais tabulares monoespaçados (`font-mono tabular-nums`) para zero trepidação horizontal durante atualizações rápidas; unidades (`km/h`, `m`, `s`, `deg`) ficam visualmente subordinadas ao valor principal.
  - *Ergonomia Mobile & Presença Desktop*:
    - No celular (`375px–430px`), todos os botões interativos respeitam área mínima de toque de `44×44px`, mantendo barras fixas dentro do teto ergonômico compacto e controles críticos ao alcance do polegar.
    - No desktop (`1440px`), o layout se expande em um console assimétrico de múltiplas colunas (coluna esquerda de instrumentos primitivos + cronômetro/calibração VSS, palco central amplo com o Radar Vetorial Offline e faixa superior de status/controles), aproveitando 100% da área útil sem encolher em um cartão estreito.

- **Feedback Interativo & Estados Claros**:
  - Estados explícitos em texto claro + ícone para permissão pendente, aguardando satélites, sinal fixado (`±Xm`), permissão negada no navegador (com instrução direta de desbloqueio ou uso imediato do Simulador) e modo Simulador ativo.
  - Transições instantâneas (`≤ 150ms`) restritas a `transform` e `opacity`, mantendo o Canvas 2D desacoplado do ciclo pesado de re-renderização da árvore principal.

---

## 3. Decisões de Produto & Trade-Offs

- **Decisão 1: Isolamento Cirúrgico do Fluxo GPS por Serviço/Hook Dedicado + Referência Rápida**
  - *Abordagem Escolhida*: Manter o receptor GPS (e o simulador) em um gerenciador dedicado cuja leitura mais recente é espelhada em uma referência leve em memória (`ref`) consumida pelo laço já existente de gravação de Datalog, sem alterar uma única linha do decodificador binário de 139 bytes da ECU.
  - *Por quê*: Garante risco zero de quebrar a comunicação serial Web Serial/Bluetooth/Tauri, os mapas VE/Ignição/AFR ou o analisador de sincronismo, atendendo rigorosamente à diretriz cirúrgica.
  - *Alternativas Descartadas*: Injetar canais falsos dentro do pacote binário da ECU — descartado porque misturaria canais de hardware com sensores do smartphone e poderia afetar seletores de configuração da ECU.

- **Decisão 2: Radar Vetorial Offline em Canvas 2D com Projeção Métrica Local**
  - *Abordagem Escolhida*: Converter coordenadas geográficas `(lat, lon)` relativas ao ponto inicial da sessão em coordenadas métricas planas locais `(x, y em metros)` via projeção equiretangular local, renderizando o circuito em Canvas 2D de alta densidade (`devicePixelRatio`) com rampa de cores contínua por canal selecionado.
  - *Por quê*: Funciona sem internet em pistas isoladas, não exige chaves de API externas, consome pouquíssima bateria e permite colorir cada trecho da pista exatamente pela Velocidade, Mistura AFR, Pressão Turbo/MAP ou RPM daquele instante.
  - *Alternativas Descartadas*: Mapas baseados em blocos de rua online — falham sem cobertura 4G/5G na pista e dificultam a leitura limpa do traçado de telemetria.

- **Decisão 3: Filtro Anti-Deriva Parado e Interpolação Linear no Cronômetro de Arrancada**
  - *Abordagem Escolhida*: Ignorar micro-deslocamentos quando a velocidade GPS for inferior a `1.8 km/h` ou quando o deslocamento for menor que o raio de incerteza instantâneo para o cálculo do Odômetro *Trip*, e aplicar interpolação temporal linear entre amostras consecutivas ao cruzar `60 km/h`, `100 km/h`, `200 km/h`, `201m` e `402m`.
  - *Por quê*: Evita que o odômetro some distância fantasma com o carro parado no box e melhora drasticamente a repetibilidade das parciais mesmo em receptores GPS internos de `1–5 Hz`.

- **Decisão 4: Compatibilidade Nativa com MegaLogViewer nos Arquivos `.msl`**
  - *Abordagem Escolhida*: Quando o GPS estiver ativo durante qualquer quadro da gravação, acrescentar automaticamente ao `.msl` as colunas com os rótulos reconhecidos nativamente pelo MegaLogViewer (`GPS Speed`, `Latitude`, `Longitude`, `Altitude`, `Heading`, `GPS Accuracy`, `GPS Gear`, `GPS Trip`), preservando 6 casas decimais em `Latitude`/`Longitude` (~11 cm de resolução espacial). Quando o GPS nunca tiver sido ativado na gravação, o `.msl` permanece inalterado.
  - *Por quê*: Permite abrir o log diretamente na visão de mapa/pista do MegaLogViewer sem precisar renomear colunas manualmente, sem poluir logs feitos em dinamômetro ou bancada com colunas zeradas quando o GPS está desligado.

---

## 4. Arquitetura Técnica & Estratégia de Dados *(Referência Técnica)*

- **Diagrama de Arquitetura & Fluxo de Dados**:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     CAMADA DE AQUISIÇÃO GPS & BANCADA                        │
│  ┌──────────────────────────────┐      ┌──────────────────────────────────┐  │
│  │ Browser Geolocation API      │      │ Simulador de Rota GPS (Bancada)  │  │
│  │ watchPosition (HighAccuracy) │      │ • Arrancada 402m                 │  │
│  │ + Screen Wake Lock API       │      │ • Circuito Track Day             │  │
│  └──────────────┬───────────────┘      │ • Cruzeiro Aferição VSS          │  │
│                 └──────────┬───────────┴─────────────────┬────────────────┘  │
└────────────────────────────┼─────────────────────────────┼───────────────────┘
                             ▼                             ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                  MOTOR DE TELEMETRIA GPS & PERFORMANCE                       │
│  • Filtro anti-deriva + Odômetro Haversine (Trip km, Vmax, Ganho Altitude)   │
│  • Estimador de Marcha (km/h por 1000 RPM) + Comparador GPS vs ECU VSS       │
│  • Cronômetro Automático (0-60, 0-100, 100-200 km/h, 60ft, 201m, 402m)       │
│  • Buffer Circular de Trajeto 2D (x/y metros + Speed, AFR, MAP, RPM)         │
└──────────────┬───────────────────────────────────────────┬───────────────────┘
               │ Snapshot em Memória (Ref Síncrona)        │ Estado Reativo UI
               ▼                                           ▼
┌──────────────────────────────────────┐  ┌────────────────────────────────────┐
│   GRAVADOR DE DATALOG (.MSL) ATUAL   │  │  COCKPIT DEDICADO GPS & TRACK      │
│  • Lê pacotes 139B da ECU intactos   │  │  • Aba "GPS" na Barra Mobile       │
│  • Se GPS ativo: anexa canais GPS    │  │  • Aba "GPS & Track" no Desktop    │
│    padronizados MegaLogViewer        │  │  • Canvas 2D Radar Vetorial        │
│  • Se GPS inativo: saída original    │  │  • Painéis Drag Timer & Calib VSS  │
└──────────────────────────────────────┘  └────────────────────────────────────┘
```

- **Modelo de Dados & Estado**:
  - **Amostra Instantânea GPS**: `speedKmh`, `latitude`, `longitude`, `altitudeM`, `headingDeg`, `accuracyM`, `updateRateHz`, `source` (`"hardware"` | `"simulator"` | `"idle"`), `permissionState` e `timestampMs`.
  - **Métricas Acumuladas de Sessão**: `tripDistanceKm`, `maxSpeedKmh`, `minAltitudeM`, `maxAltitudeM`, `estimatedGear` (`0..6`), `vssDeltaKmh`, `vssErrorPct`, `suggestedPulsesPerKm`, e histórico de pontos projetados para o radar vetorial (limitado a buffer circular eficiente para sessões longas).
  - **Estado do Cronômetro de Performance**: máquina de estados determinística (`idle` → `armed` → `running` → `completed`) armazenando marcas de tempo e velocidades de passagem para cada meta de velocidade/distância, além da lista de puxadas anteriores da sessão.
  - **Calibração Local de Relações de Marcha**: vetor de 6 relações (`km/h por 1000 RPM`) persistido localmente no navegador, inicializado automaticamente com os valores de `VSS and Gear Detection` da ECU quando presentes ou com escalonamento padrão editável com 1 toque.

- **Mapeamento de Componentes Interativos & Tratamento de Eventos**:
  - **Controle Fonte GPS (`Iniciar GPS Real` / `Parar` / `Simulador`)**: Inicia ou encerra `watchPosition({ enableHighAccuracy: true, maximumAge: 0, timeout: 10000 })`, limpando o `watchId` com segurança ao desmontar ou alternar para o simulador; trata erros de permissão/timeout exibindo orientação clara na própria tela.
  - **Controle `Tela Acesa` (*Wake Lock*)**: Solicita `navigator.wakeLock.request("screen")` quando suportado, reativa automaticamente ao retornar a aba à visibilidade (`visibilitychange`) e exibe confirmação real do estado da trava.
  - **Radar Vetorial Interativo (Canvas)**: Suporta seleção do canal de cor (`Velocidade`, `AFR`, `MAP`, `RPM`), alternância `Norte Acima` / `Frente Acima`, botões `+` / `-` / `Centralizar`, zoom por roda/pinça, arraste panorâmico e inspeção por toque/cursor mostrando velocidade, RPM e AFR exatos do ponto clicado no traçado.
  - **Painel Drag Timer**: Botões `Armar Puxada`, `Zerar Cronômetro` e `Limpar Histórico`, com transições automáticas baseadas na velocidade/distância acumulada desde o disparo.
  - **Painel Marcha & VSS**: Botões `Aprender M1..M6` (capturam `GPS Speed / (RPM / 1000)` quando `RPM > 800` e `GPS Speed > 5 km/h`), botão `Restaurar Padrão` e leitura direta do desvio frente ao `VSS` da ECU com cálculo de `Pulses/km` corrigido.
  - **Integração com Gravação de Log**: Botão direto `Gravar Log` / `Parar Log` conectado ao mesmo controlador central de Datalog da aplicação, exibindo indicador visual claro quando os canais de GPS estão sendo injetados no arquivo `.msl`.
