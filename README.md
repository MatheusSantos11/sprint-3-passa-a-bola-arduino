# Projeto IoT — Monitoramento de Partida de Futebol

=========================================================
			Nome dos integrantes
------------------------------------------------
Henrique de Oliveira Gomes           | RM566424

Henrique Kolomyes Silveira           | RM563467

Matheus Santos de Oliveira           | RM561982

Vinicius Alexandre Aureliano Ribeiro | RM561606

------------------------------------------------
================================================

## 1. Introdução
Este repositório contém a implementação de um sistema de monitoramento de partida de futebol, composto por três subsistemas principais: um dispositivo ESP32 simulado (Wokwi), um simulador de telemetria em Python e um painel de visualização em Node‑RED. A comunicação entre componentes é realizada via protocolo MQTT.

## 2. Componentes
- **ESP32 (Wokwi)**  
  - Publica dados de posse de bola no tópico `futebol/posse`.  
  - Publica eventos de gol no tópico `futebol/gol`.  
  - Utiliza LEDs para indicação visual dos estados: verde (Time A com maior posse), azul (Time B com maior posse) e vermelho (pisca no evento de gol).

- **Simulador Python (`device_sim.py`)**  
  - Gera e publica telemetria de jogadores (batimentos cardíacos e velocidade) nos tópicos `futebol/jogador01/telemetry` e `futebol/jogador02/telemetry`.

- **Node‑RED + Dashboard**  
  - Consome mensagens MQTT a partir de um broker público (`test.mosquitto.org:1883`) e apresenta as informações em um painel web (gauges e elementos de texto).

## 3. Arquitetura de Comunicação
O sistema utiliza um broker MQTT público como elemento de integração. Os publishers (ESP32 e script Python) publicam mensagens JSON em tópicos definidos; o Node‑RED subscreve esses tópicos e realiza o processamento/visualização.

## 4. Estrutura dos tópicos MQTT
| Tópico | Origem | Formato do payload |
|---|---:|---|
| `futebol/posse` | ESP32 | `{"timeA": <int>, "timeB": <int>}` |
| `futebol/gol` | ESP32 | `{"time": "A" | "B", "distancia": <int>}` |
| `futebol/jogadorXX/telemetry` | Python | `{"player": "<id>", "heart_rate": <int>, "speed": <float>}` |

## 5. Instruções de implantação

### 5.1 Requisitos
- Node.js e npm (para Node‑RED)
- Python 3.7+ (para o simulador)
- Biblioteca Python: `paho-mqtt`
- (Opcional) Wokwi para simulação do ESP32

### 5.2 Execução do ESP32 (Wokwi)
1. Acesse o ambiente Wokwi em https://wokwi.com.  
2. Importe/cole o código fonte do ESP32.  
3. Configure os pinos de LED conforme necessário (pinos 2, 4 e 5).  
4. Inicie a simulação e observe as mensagens publicadas no console serial.

### 5.3 Execução do simulador Python
Instale a dependência:
```bash
pip install paho-mqtt
```
Execute o simulador:
```bash
python device_sim.py
```
O script publica telemetria de jogadores periodicamente (a cada 3 segundos, por padrão).

### 5.4 Execução do Node‑RED e importação do fluxo
Instale Node‑RED (caso não esteja instalado):
```bash
npm install -g node-red
```
Instale o pacote de dashboard:
```bash
npm install node-red-dashboard
```
Proceda:
1. Inicie o Node‑RED:
```bash
node-red
```
2. Acesse o editor em `http://localhost:1880`.  
3. Importe o arquivo `flows.json` disponibilizado neste repositório.  
4. Verifique a configuração do nó MQTT Broker para `test.mosquitto.org:1883`.  
5. Acesse o painel em `http://localhost:1880/ui`.

## 6. Componentes do painel (Node‑RED Dashboard)
- **Gauge — Batimento Cardíaco**: exibe o valor de batimentos por minuto (bpm) dos jogadores.
- **Gauge — Time A / Time B**: exibe a porcentagem de posse de bola.
- **UI Text / Notificação**: exibe mensagens de gol com indicação de time e distância.

## 7. Solução de problemas
- **Dados não aparecem no dashboard**: confirme se o broker configurado no Node‑RED é o mesmo utilizado pelos publishers.  
- **JSON tratado como string no Node‑RED**: ajuste a propriedade `datatype` do nó MQTT in para `json`.  
- **ESP32 não conecta (Wokwi)**: verifique as configurações de rede do simulador (SSID `Wokwi-GUEST`).

## 8. Possíveis aprimoramentos
- Persistência de eventos (ex.: armazenamento em banco de dados SQLite ou InfluxDB).  
- Substituição do broker público por um broker local (Mosquitto) para uso em ambiente restrito.  
- Implementação de histórico temporal (charts) para análise de posse ao longo do tempo.  
- Integração com front‑end externo (React, Vue) para disponibilização pública.

## 9. Licença
Este projeto encontra‑se disponível sob a licença MIT (ou outra de sua preferência). Consulte o arquivo `LICENSE` caso deseje aplicar uma licença específica.

---

Se desejar, realizo a tradução deste documento para inglês, ou então preparo um `docker-compose` com Mosquitto e Node‑RED para execução local e consistente do ambiente.
