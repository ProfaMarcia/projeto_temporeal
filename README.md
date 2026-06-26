# MonoxGuard-RT

Detector de monóxido de carbono (CO) embarcado em **ESP32 + FreeRTOS**, desenvolvido como
estudo de caso de **Sistemas de Tempo-Real** (disciplina PGENE537 — PPGEE/UFAM).

O sistema lê um sensor de gás, processa as leituras com filtragem e histerese, e aciona
alarmes (LED, buzzer, relé) e um display LCD quando a concentração de CO ultrapassa limiares
de segurança — sempre dentro de um *deadline* temporal garantido.

> **Nota sobre a plataforma:** o projeto foi originalmente concebido para hardware físico
> (ESP32 + sensor MQ-7). Após a indisponibilidade da placa, os experimentos foram conduzidos
> no simulador **Wokwi**, executando o mesmo firmware, com o sinal do sensor emulado por um
> potenciômetro na entrada ADC. A arquitetura de tarefas e a lógica avaliada são idênticas.

---

## Arquitetura

O firmware é organizado em três camadas, mais tarefas concorrentes do FreeRTOS:

```
src/
├── core/         Lógica pura (verificável e testável isoladamente)
│   ├── calib.*       Conversão ADC -> ppm (aritmética 64-bit, com clamp)
│   ├── detection.*   Máquina de estados NORMAL/WARN/ALARM (histerese + debounce)
│   └── filter.*      Filtro de média móvel circular (janela N=8)
├── hal/          Abstração de hardware (sensor, atuadores)
├── tasks/        8 tarefas FreeRTOS (ver tabela abaixo)
├── metrics/      Instrumentação temporal (timing.*)
├── config.h      Parâmetros: períodos, prioridades, limiares, deadline
├── shared.h      Estado compartilhado entre tarefas
└── main.cpp      setup() + criação das tarefas
```

### Modelo de tarefas

| Tarefa     | Período    | Prioridade | Tipo                |
|------------|------------|------------|---------------------|
| alarme     | esporádica | 6 (máx.)   | reativa (crítica)   |
| sensor     | 100 ms     | 5          | periódica           |
| detecção   | por evento | 4          | dirigida a evento   |
| botão      | esporádica | 3          | reativa (ISR)       |
| telemetria | 1000 ms    | 2          | periódica           |
| display    | 300 ms     | 2          | periódica           |
| DHT11      | 2000 ms    | 1 (mín.)   | periódica (carga)   |
| watchdog   | 2000 ms    | 1 (mín.)   | periódica           |

Escalonamento por **prioridade fixa (Rate-Monotonic)** em núcleo único. Deadline crítico do
alarme: **50 ms**.

---

## Como rodar (Wokwi + PlatformIO no VS Code)

1. Instale o **PlatformIO** e a extensão **Wokwi** no VS Code.
2. Compile o firmware (no terminal, **não** use o botão de debug):
   ```bash
   pio run
   ```
3. Inicie a simulação: `F1` → **Wokwi: Start Simulator**.
4. Gire o potenciômetro do gás para variar a concentração de CO e observe os estados
   (NORMAL → WARN → ALARM), o LCD, o LED e o buzzer.
5. O botão **MUTE** silencia o alarme sonoro.

A telemetria é emitida pela serial em formato CSV (ver seção abaixo).

---

## Resultados experimentais (resumo)

| Métrica                  | Valor medido        | Referência              |
|--------------------------|---------------------|-------------------------|
| Latência do alarme       | 89–90 µs            | deadline 50 000 µs      |
| WCET (tarefa detecção)   | 37 µs               | período 100 ms          |
| Latência do botão (MUTE) | ≈27 ms              | inclui debounce de 30 ms|
| Jitter de período        | 0 µs                | idealizado (simulação)  |
| Heap livre               | ≈312 kB estável     | sem vazamento           |
| Amostras descartadas     | 0                   | —                       |

**Análise de escalonabilidade:** utilização total da CPU ≈ **0,43%**, muito abaixo do limite
de Liu & Layland para 6 tarefas (≈ 73,48%). Conjunto **garantidamente escalonável** por
Rate-Monotonic; todos os tempos de resposta (RTA) abaixo dos respectivos deadlines.

---

## Estrutura do repositório

```
.
├── src/                  Firmware (ver arquitetura acima)
├── verification/         Harnesses de verificação formal (trabalho irmão, PGENE601)
├── wokwi/                Diagrama do circuito (diagram.json)
├── experiments/          Dados e scripts de análise
│   ├── telemetria_log_final.csv    Log bruto de telemetria (dados dos experimentos)
│   ├── sched_analysis.py           Simulador de escalonabilidade (U, Liu&Layland, RTA, Gantt)
│   └── parse_telemetry.py          Geração dos gráficos a partir do log
├── docs/                 Artigo e figuras
├── diagram.json          Circuito do Wokwi (raiz)
├── platformio.ini        Configuração do PlatformIO
└── README.md
```

### Formato do CSV de telemetria

```
t_ms,raw,ppm,state,sensor_jitter_us,detect_jitter_us,alarm_latency_us,
button_latency_us,dropped,temp_c,humidity,detect_exec_us,detect_exec_max_us
```

### Reproduzir a análise

```bash
# requisitos: python3 + matplotlib + pandas
pip3 install matplotlib pandas    # (ou: sudo apt install python3-matplotlib python3-pandas)

python3 experiments/sched_analysis.py     # análise de escalonabilidade + Gantt
python3 experiments/parse_telemetry.py    # gráficos da telemetria
```

---

## Parâmetros principais (`src/config.h`)

| Parâmetro          | Valor    | Significado                          |
|--------------------|----------|--------------------------------------|
| `WARN_PPM`         | 300      | limiar de atenção                    |
| `ALARM_PPM`        | 800      | limiar de alarme                     |
| `HYST_PPM`         | 50       | histerese (evita chattering)         |
| `DEBOUNCE_N`       | 3        | amostras p/ confirmar transição      |
| `FILTER_WINDOW`    | 8        | janela do filtro de média móvel      |
| `ALARM_DEADLINE_US`| 50000    | deadline do alarme (50 ms)           |

---

## Disciplina

PGENE537 — Sistemas de Tempo-Real • PPGEE/UFAM • Orientador: Prof. Lucas Cordeiro

**Autores:** Bruno Gonçalves, Dioneide Sales, Márcia Silva.

> Observação: a calibração ADC→ppm usa um mapeamento linear simplificado, não uma curva de
> calibração real do sensor MQ-7. Para uso em campo, seria necessária calibração com gás de
> referência.
