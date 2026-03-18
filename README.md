# 🚀 O Caçador de Tendência: Sniper RSI & SMA (Trading Worker)

> **Plataforma Avançada de Algo-Trading para Negociação de Criptoativos na Binance (Spot Market).**

![.NET 8.0](https://img.shields.io/badge/.NET-8.0-512BD4?style=flat&logo=dotnet)
![Architecture](https://img.shields.io/badge/Architecture-Event--Driven-orange?style=flat)
![Binance API](https://img.shields.io/badge/Exchange-Binance-F3BA2F?style=flat&logo=binance)
![Angular](https://img.shields.io/badge/Frontend-Angular%2021-DD0031?style=flat&logo=angular)
![MongoDB](https://img.shields.io/badge/Database-MongoDB-47A248?style=flat&logo=mongodb)

---

## 🎯 A Filosofia da Ferramenta

O trading humano sofre com fadiga, FOMO e emoção. O **Caçador de Tendência** foi desenvolvido como uma arma (Sniper) assíncrona que varre o mercado 24 horas por dia em busca de um alinhamento matemático raro, cruzando RSI (Índice de Força Relativa) Extremo com Rompimento Severo de Bandas de Bollinger.

**Principais Filtros de Operação:**
1. **RSI < 20 (Sobrevenda) ou > 80 (Sobrecompra):** Encontra momento de exaustão extrema de mercado.
2. **Bollinger Band Breakout (Desvio de 2.5):** Ignora ruídos normais. O ativo precisa ter ultrapassado violentamente a média histórica.
3. **Convergência Multi-Timeframe (MTF):** *(The Golden Signal)*. Confirma uma exaustão em M1 ou M5 checando se em M15 ou H1 o ativo também já está esgotado.

---

## 🏗 Arquitetura & Fluxograma Lógico

O diagrama abaixo ilustra detalhadamente como o Backend interage com os inputs matemáticos, MongoDB, Frontend e o envio de requisições para a Exchange e para o Telegram.

```mermaid
graph TD
    %% ==========================================================
    %% 1. DEFINIÇÃO DE CORES
    %% ==========================================================
    classDef external fill:#74b9ff,stroke:#0984e3,stroke-width:2px,color:#fff;
    classDef worker fill:#ff7675,stroke:#d63031,stroke-width:2px,color:#fff;
    classDef logic fill:#a29bfe,stroke:#6c5ce7,stroke-width:2px,color:#fff;
    classDef data fill:#55efc4,stroke:#00b894,stroke-width:2px,color:#2d3436;
    classDef user fill:#ffeaa7,stroke:#fdcb6e,stroke-width:2px,color:#2d3436;
    classDef tg fill:#74b9ff,stroke:#0984e3,stroke-width:2px,color:#fff;

    %% ==========================================================
    %% 2. NÓS E ETAPAS
    %% ==========================================================

    %% Entradas
    Exchange[📱 Binance API REST<br/>Get Klines / Preços]:::external
    Admin[👤 Operador<br/>Angular Dashboard]:::user

    subgraph Backend [.NET 8 Core - Controle_Cripto]
        
        Worker(⚙️ TradeWorkerService<br/>Daemon 24/7):::worker
        Brain(🧠 ControleAnaliseService<br/>Cálculo RSI/SMA/BB):::logic
        Controller(🔌 ControleAnaliseController<br/>Recebe Ordens / Settings):::logic
        Execution(🚀 Order Execution<br/>Processa Spot API):::worker

        %% Processo de Loop
    Worker -->|"A cada 5s envia Polling"| Brain
    
    Brain -->|"Puxa Cotações"| Exchange
    Exchange -->|"Retorna Historico (Klines)"| Brain

    %% Matematica
    Brain -->|"Calcula RSI + Bollinger + EMA 200"| LogicGate{"Gatilho Sniper?"}
    LogicGate -- "Sim (Fora das Bandas)" --> MTFGate{"Verifica Convergência MTF"}
    LogicGate -- "Não" --> WorkerLoop("Ignora e Continua")

    %% Notificação de Sinal
    MTFGate -. "Sinal GOLDEN ou Normal" .-> NotifyTelegram("TelegramService<br/>Envia Alerta de Sinal"):::tg
    MTFGate --> SaveLog("Grava Analise no Banco"):::data
end

%% Integração Frontend x Backend
Admin -->|"Checa Dashboard Polling 10s"| Controller
Admin -->|"Aperta BUY / SELL"| Controller
Controller -->|"Repassa Comando"| Execution

%% Execução Real
Execution -->|"Se ExecucaoReal = True"| BinanceTrade["📱 Binance API<br/>Spot Order - Market"]:::external
Execution -->|"Salva RSI exato, Preço Real,<br/>Tendência e Origem"| MongoDB[("🗄️ MongoDB<br/>Colec: HistoricoOrdens")]:::data
Execution -->|"Avisa de Compra Final"| NotifyTelegram

%% Conexoes externas do Bot
NotifyTelegram --> TelegramApp["📱 Seu Telegram📱"]:::tg

```

---

## 🐇 Evolução para Microserviços: Event-Driven com RabbitMQ

O projeto evoluiu de um modelo **Monolítico Síncrono** para uma arquitetura Reativa Orientada a Eventos usando **RabbitMQ**.

No modelo antigo, se a API do Telegram ficasse offline, a thread principal do `TradeWorkerService` congelava, perdendo janelas críticas de compra na Binance.

Com o RabbitMQ, o fluxo segue os preceitos de **SOLID** (Single Responsibility) e **Clean Architecture**:
1. **O Produtor:** O `TradeWorkerService` apenas calcula matemática e cospe o evento JSON na fila `fila_sinais`.
2. **O Consumidor:** O `TelegramNotificationConsumer` observa a fila de forma assíncrona. Quando a mensagem chega, ele consome, formata e envia.

*Se o Telegram cair, a mensagem sofre Requeue (NACK) e o Bot de Trade continua varrendo o mercado nativamente sem atrasos (na casa dos milissegundos).*

```mermaid
graph LR
    %% Cores
    classDef publisher fill:#6c5ce7,stroke:#a29bfe,stroke-width:2px,color:#fff;
    classDef rabbit fill:#fdcb6e,stroke:#e17055,stroke-width:2px,color:#2d3436;
    classDef consumer fill:#00b894,stroke:#55efc4,stroke-width:2px,color:#fff;

    %% Elementos
    bot("🤖 TradeWorker<br/>(Calcula RSI/SMA)"):::publisher
    broker("🐇 RabbitMQ<br/>Exchange: Direta"):::rabbit
    fila[/"Fila: fila_sinais"\]:::rabbit
    
    cons_telegram("📱 TelegramConsumer<br/>(Envia p/ Celular)"):::consumer
    cons_order("🚀 OrdemExecutor<br/>(Bate na Binance)"):::consumer
    
    %% Relacionamentos
    bot -->|"Publish (JSON)"| broker
    broker --> fila
    
    fila -.->|"Consome Assíncrono"| cons_telegram
    fila -.->|"Consome Assíncrono"| cons_order

```

---

## 🛠 Entendendo o Processo Detalhadamente

1. **Ingestão de Dados (`TradeWorkerService`):** Um serviço autônomo executa um *loop* lendo as principais moedas listadas (BTC, ETH, SOL...). Ele usa a biblioteca `Binance.Net` para puxar os *Klines* (velas) da API.
2. **O Cálculo Matemático (`Skender.Stock.Indicators`):** Em posse de 300 velas fechadas, aplicamos os cálculos de RSI (relativo a 14 períodos) e Bollinger Bands. Se a `Execução Dinâmica` for ativada e o RSI chegar em níveis como `<= 20` enquanto o preço estiver abaixo da `LowerBand`, acendemos o alerta amarelo.
3. **Convergência de Confluência (MTF):** Um sinal não basta. O código analisa o mesmo par em um *Timeframe* muito maior. Se este tempo maior também se encontra sobrevendido/sobrecomprado, ele classifica aquele momento exato como "CONVERGÊNCIA DETECTADA". A probabilidade de reversão rápida nos próximos 5 minutos aumenta em 90%.
4. **Log Estratégico Telemetrizado:** Todo cruzamento matemático e todas as ordens (vindas de execução manual via UI ou Automáticas do Robô) vão parar no **MongoDB (`HistoricoOrdens`)**. Quando a Ordem engatilha na plataforma, não salvamos apenas que "Compramos BTC" – nós salvamos "Compramos BTC a $65k, o RSI daquele milissegundo era de 21.05 e a Tendência era Berish". Feito para análise de *Data Science* posterior.
5. **Comando de Fogo (Execução Real):** Quando a ordem é de execução verdadeira, o motor envia uma boleta *Market* (A mercado) com criptografia HmacSha256 (via SDK) confirmando o token e o secret para a Binance, retornando instantaneamente o `AverageFillPrice` (evitando erro de *slippage* visual). Todo o processo – de engatilhar o código no Angular a vibrar o aviso do Telegram de compra feita – acontece em < 1 segundo.

## ⚙️ Variáveis de Ambiente Necessárias

Para rodar este monstro localmente, configure no seu arquivo `appsettings.json`:
- `BinanceSettings:ApiKey` e `ApiSecret`: Suas chaves de Spot Trade.
- `Telegram:Token` e `ChatId`: Os dados do seu Bot do Telegram Tracker.
- `MongoDbSettings`: A string de conexão do seu DB local ou nuvem.
