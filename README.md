# 
flowchart LR
  %% ====== DATA + STATE ======
  subgraph D[Data & State Layer]
    MD[(Market Data\n(Ticks/Bars, Calendar, Session Times))]
    AD[(Account Data\nBalance, Unrealized, Peak Equity,\nTrailing Threshold Distance)]
    FS[(Feature Store / Blackboard\nVWAP, ATR, ORH/ORL,\nKey Levels, Regime, Limits)]
    DB[(Journal + Metrics DB)]
  end

  %% ====== ORCHESTRATION ======
  subgraph O[Orchestration Layer]
    ORCH[Orchestrator / State Machine\nPREP→READY→ARMED→IN_TRADE→COOLDOWN→READY\nSTOPPED / FLATTEN]
    BUS[[Event Bus\n(bar_close, signal, fill, risk_alert, time_tick)]]
  end

  %% ====== AGENTS ======
  subgraph A[Agents (Propose / Analyze)]
    REG[Regime Agent\n(trend / range / chop)]
    SET1[Setup Agent: ORB-15]
    SET2[Setup Agent: Trend Pullback]
    SET3[Setup Agent: VWAP Mean Reversion (optional)]
    SCORE[Quality Scorer\n(0–100 + reasons)]
    RISK[Risk & Sizing Agent\n(stop ticks → contracts\nNQ/ES or micros)]
    REVIEW[Review Agent\n(weekly analysis,\nparameter suggestions)]
  end

  %% ====== GOVERNORS (VETO) ======
  subgraph G[Governors (Deterministic VETO)]
    COMP[Compliance Governor (Apex Rules)\n- Time-to-flat gate\n- Trailing threshold buffer gate\n- Daily stop / profit cap\n- Max open risk]
    KILL[Kill Switch\nFlatten + Disable Trading]
  end

  %% ====== EXECUTION ======
  subgraph X[Execution Layer]
    EXEC[Execution Engine\n(order router + brackets)]
    BROKER[(Sim / Live Bridge\n(Rithmic / Broker API))]
    MON[Monitoring + Alerts\n(Peak equity, drawdown,\nslippage, data health)]
  end

  %% ====== FLOWS ======
  MD --> FS
  AD --> FS
  FS --> BUS
  BUS --> ORCH

  ORCH --> REG --> FS
  ORCH --> SET1 --> FS
  ORCH --> SET2 --> FS
  ORCH --> SET3 --> FS

  FS --> SCORE --> RISK --> COMP

  COMP -- approve --> EXEC --> BROKER
  COMP -- veto --> ORCH
  COMP -- danger --> KILL --> EXEC

  BROKER --> MON --> FS
  FS --> DB
  DB --> REVIEW --> FS

  %% ====== NOTES ======
  %% Compliance Governor encodes:
  %%  - Flat by 4:59pm ET
  %%  - Trailing threshold follows peak unrealized equity until safety net
