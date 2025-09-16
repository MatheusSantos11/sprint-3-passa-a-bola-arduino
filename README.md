# Projeto IoT — Monitoramento de Partida de Futebol

---
    Henrique de Oliveira Gomes           | RM566424
    Henrique Kolomyes Silveira           | RM563467
    Matheus Santos de Oliveira           | RM561982
    Vinicius Alexandre Aureliano Ribeiro | RM561606

---

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
 
# Link do video do youtube 

https://youtu.be/HnyHcioT-vk

# Codigo de Arduino
link de arduino: https://wokwi.com/projects/441818652069016577

#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>

// --- CONFIG ---
const char* WIFI_SSID = "Wokwi-GUEST";
const char* WIFI_PASS = "";
const char* MQTT_BROKER = "test.mosquitto.org"; // outro broker público
const uint16_t MQTT_PORT = 1883;

WiFiClient espClient;
PubSubClient client(espClient);

// Pinos dos LEDs
const int LED_A = 2;   // verde -> time A tem mais posse
const int LED_B = 4;   // azul -> time B tem mais posse
const int LED_GOL = 5; // vermelho -> pisca em caso de gol

unsigned long lastUpdate = 0;

void connectWiFi() {
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  Serial.print("Conectando ao WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println(" conectado!");
}

void reconnectMQTT() {
  while (!client.connected()) {
    Serial.print("Conectando ao MQTT...");
    if (client.connect("esp32-indicator")) {
      Serial.println(" conectado!");
    } else {
      Serial.print(" falhou, rc=");
      Serial.print(client.state());
      delay(2000);
    }
  }
}

void publicarPosseEGol() {
  // Simula posse de bola
  int posseA = random(40, 61);
  int posseB = 100 - posseA;

  // LED verde ou azul dependendo de quem tem mais posse
  if (posseA > posseB) {
    digitalWrite(LED_A, HIGH);
    digitalWrite(LED_B, LOW);
  } else {
    digitalWrite(LED_A, LOW);
    digitalWrite(LED_B, HIGH);
  }

  // Publica no MQTT
  StaticJsonDocument<128> doc;
  doc["timeA"] = posseA;
  doc["timeB"] = posseB;
  char buffer[128];
  size_t n = serializeJson(doc, buffer);
  client.publish("futebol/posse", buffer, n);

  Serial.printf("[Posse] TimeA=%d%% | TimeB=%d%%\n", posseA, posseB);

  // Chance de gol
  if (random(0, 100) < 20) { // 20% de chance
    StaticJsonDocument<128> golDoc;
    String time = (random(0, 2) == 0) ? "A" : "B";
    int distancia = random(10, 36);
    golDoc["time"] = time;
    golDoc["distancia"] = distancia;
    char buf[128];
    size_t n2 = serializeJson(golDoc, buf);
    client.publish("futebol/gol", buf, n2);

    Serial.printf("⚽ GOL do Time", time.c_str(), distancia);

    // Pisca LED vermelho
    for (int i = 0; i < 5; i++) {
      digitalWrite(LED_GOL, HIGH);
      delay(200);
      digitalWrite(LED_GOL, LOW);
      delay(200);
    }
  }
}

void setup() {
  pinMode(LED_A, OUTPUT);
  pinMode(LED_B, OUTPUT);
  pinMode(LED_GOL, OUTPUT);
  Serial.begin(115200);
  connectWiFi();
  client.setServer(MQTT_BROKER, MQTT_PORT);
}

void loop() {
  if (!client.connected()) {
    reconnectMQTT();
  }
  client.loop();

  if (millis() - lastUpdate > 5000) { // a cada 5 segundos
    lastUpdate = millis();
    publicarPosseEGol();
  }
}


# Codigo de python

import time
import json
import random
import paho.mqtt.client as mqtt

BROKER = "broker.hivemq.com"
PORT = 1883

client = mqtt.Client()
client.connect(BROKER, PORT, 60)

def simulate_player(player_id):
    return {
        "player": player_id,
        "ts": int(time.time()),
        "heart_rate": random.randint(60, 180),   # batimentos
        "speed": round(random.uniform(5, 25), 2)  # km/h
    }

try:
    while True:
        for player_id in ["jogador01", "jogador02"]:
            payload = simulate_player(player_id)
            topic = f"futebol/{player_id}/telemetry"
            client.publish(topic, json.dumps(payload))
            print("Published:", payload)
        time.sleep(3)
except KeyboardInterrupt:
    client.disconnect()
    print("Finalizado")

# Codigo Json do node-red

[
    {
        "id": "d716c1e95bfd1b2c",
        "type": "tab",
        "label": "Fluxo 1",
        "disabled": false,
        "info": "",
        "env": []
    },
    {
        "id": "a4e1cb042701e7a7",
        "type": "mqtt-broker",
        "name": "mosquitto",
        "broker": "localhost",
        "port": 1883,
        "clientid": "",
        "autoConnect": true,
        "usetls": false,
        "protocolVersion": 4,
        "keepalive": 60,
        "cleansession": true,
        "autoUnsubscribe": true,
        "birthTopic": "",
        "birthQos": "0",
        "birthRetain": "false",
        "birthPayload": "",
        "birthMsg": {},
        "closeTopic": "",
        "closeQos": "0",
        "closeRetain": "false",
        "closePayload": "",
        "closeMsg": {},
        "willTopic": "",
        "willQos": "0",
        "willRetain": "false",
        "willPayload": "",
        "willMsg": {},
        "userProps": "",
        "sessionExpiry": ""
    },
    {
        "id": "broker1",
        "type": "mqtt-broker",
        "name": "Local Mosquitto",
        "broker": "broker.hivemq.com",
        "port": "1883",
        "clientid": "node-red",
        "autoConnect": true,
        "usetls": false,
        "protocolVersion": "4",
        "keepalive": "60",
        "cleansession": true,
        "autoUnsubscribe": true,
        "birthTopic": "",
        "birthQos": "0",
        "birthPayload": "",
        "birthMsg": {},
        "closeTopic": "",
        "closeQos": "0",
        "closePayload": "",
        "closeMsg": {},
        "willTopic": "",
        "willQos": "0",
        "willPayload": "",
        "willMsg": {},
        "userProps": "",
        "sessionExpiry": ""
    },
    {
        "id": "ui-tab",
        "type": "ui_tab",
        "name": "Dashboard Futebol",
        "icon": "dashboard",
        "order": 1,
        "disabled": false,
        "hidden": false
    },
    {
        "id": "ui-group",
        "type": "ui_group",
        "name": "Futebol",
        "tab": "ui-tab",
        "order": 2,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "3b0c2e6fb905a6d5",
        "type": "ui_base",
        "theme": {
            "name": "theme-light",
            "lightTheme": {
                "default": "#0094CE",
                "baseColor": "#0094CE",
                "baseFont": "-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Oxygen-Sans,Ubuntu,Cantarell,Helvetica Neue,sans-serif",
                "edited": true,
                "reset": false
            },
            "darkTheme": {
                "default": "#097479",
                "baseColor": "#097479",
                "baseFont": "-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Oxygen-Sans,Ubuntu,Cantarell,Helvetica Neue,sans-serif",
                "edited": false
            },
            "customTheme": {
                "name": "Untitled Theme 1",
                "default": "#4B7930",
                "baseColor": "#4B7930",
                "baseFont": "-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Oxygen-Sans,Ubuntu,Cantarell,Helvetica Neue,sans-serif"
            },
            "themeState": {
                "base-color": {
                    "default": "#0094CE",
                    "value": "#0094CE",
                    "edited": false
                },
                "page-titlebar-backgroundColor": {
                    "value": "#0094CE",
                    "edited": false
                },
                "page-backgroundColor": {
                    "value": "#fafafa",
                    "edited": false
                },
                "page-sidebar-backgroundColor": {
                    "value": "#ffffff",
                    "edited": false
                },
                "group-textColor": {
                    "value": "#1bbfff",
                    "edited": false
                },
                "group-borderColor": {
                    "value": "#ffffff",
                    "edited": false
                },
                "group-backgroundColor": {
                    "value": "#ffffff",
                    "edited": false
                },
                "widget-textColor": {
                    "value": "#111111",
                    "edited": false
                },
                "widget-backgroundColor": {
                    "value": "#0094ce",
                    "edited": false
                },
                "widget-borderColor": {
                    "value": "#ffffff",
                    "edited": false
                },
                "base-font": {
                    "value": "-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Oxygen-Sans,Ubuntu,Cantarell,Helvetica Neue,sans-serif"
                }
            },
            "angularTheme": {
                "primary": "indigo",
                "accents": "blue",
                "warn": "red",
                "background": "grey",
                "palette": "light"
            }
        },
        "site": {
            "name": "Node-RED Dashboard",
            "hideToolbar": "false",
            "allowSwipe": "false",
            "lockMenu": "false",
            "allowTempTheme": "true",
            "dateFormat": "DD/MM/YYYY",
            "sizes": {
                "sx": 48,
                "sy": 48,
                "gx": 6,
                "gy": 6,
                "cx": 6,
                "cy": 6,
                "px": 0,
                "py": 0
            }
        }
    },
    {
        "id": "495617585b3be31c",
        "type": "mqtt-broker",
        "name": "",
        "broker": "localhost",
        "port": 1883,
        "clientid": "",
        "autoConnect": true,
        "usetls": false,
        "protocolVersion": 4,
        "keepalive": 60,
        "cleansession": true,
        "autoUnsubscribe": true,
        "birthTopic": "",
        "birthQos": "0",
        "birthRetain": "false",
        "birthPayload": "",
        "birthMsg": {},
        "closeTopic": "",
        "closeQos": "0",
        "closeRetain": "false",
        "closePayload": "",
        "closeMsg": {},
        "willTopic": "",
        "willQos": "0",
        "willRetain": "false",
        "willPayload": "",
        "willMsg": {},
        "userProps": "",
        "sessionExpiry": ""
    },
    {
        "id": "mqtt-broker-hivemq",
        "type": "mqtt-broker",
        "name": "HiveMQ Public Broker",
        "broker": "test.mosquitto.org",
        "port": "1883",
        "clientid": "",
        "autoConnect": true,
        "usetls": false,
        "protocolVersion": 4,
        "keepalive": 15,
        "cleansession": true,
        "autoUnsubscribe": true,
        "birthTopic": "",
        "birthQos": "0",
        "birthPayload": "",
        "birthMsg": {},
        "closeTopic": "",
        "closePayload": "",
        "closeMsg": {},
        "willTopic": "",
        "willQos": "0",
        "willPayload": "",
        "willMsg": {},
        "userProps": "",
        "sessionExpiry": ""
    },
    {
        "id": "ui-tab1",
        "type": "ui_tab",
        "name": "Futebol IoT",
        "icon": "dashboard",
        "order": 2
    },
    {
        "id": "ui-group1",
        "type": "ui_group",
        "name": "Monitoramento",
        "tab": "ui-tab",
        "order": 1,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "672549255251aba1",
        "type": "mqtt-broker",
        "name": "Mosquitto Public",
        "broker": "test.mosquitto.org",
        "port": "1883",
        "clientid": "",
        "usetls": false,
        "protocolVersion": "4",
        "keepalive": "60",
        "cleansession": true,
        "birthTopic": "",
        "birthQos": "0",
        "birthPayload": "",
        "closeTopic": "",
        "closePayload": "",
        "willTopic": "",
        "willQos": "0",
        "willPayload": ""
    },
    {
        "id": "d0a22f17a1e390a4",
        "type": "mqtt in",
        "z": "d716c1e95bfd1b2c",
        "name": "Posse MQTT",
        "topic": "futebol/posse",
        "qos": "0",
        "datatype": "json",
        "broker": "672549255251aba1",
        "inputs": 0,
        "x": 230,
        "y": 300,
        "wires": [
            [
                "545b38ad9f2b4106",
                "cff91a1af2097324"
            ]
        ]
    },
    {
        "id": "545b38ad9f2b4106",
        "type": "ui_gauge",
        "z": "d716c1e95bfd1b2c",
        "name": "Time A Posse",
        "group": "ui-group",
        "order": 2,
        "width": 3,
        "height": 3,
        "gtype": "gage",
        "title": "Time A",
        "label": "%",
        "format": "{{msg.payload.timeA}}",
        "min": 0,
        "max": 100,
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "x": 490,
        "y": 280,
        "wires": []
    },
    {
        "id": "cff91a1af2097324",
        "type": "ui_gauge",
        "z": "d716c1e95bfd1b2c",
        "name": "Time B Posse",
        "group": "ui-group",
        "order": 3,
        "width": 3,
        "height": 3,
        "gtype": "gage",
        "title": "Time B",
        "label": "%",
        "format": "{{msg.payload.timeB}}",
        "min": 0,
        "max": 100,
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "x": 490,
        "y": 320,
        "wires": []
    },
    {
        "id": "b0b085f7548f800e",
        "type": "mqtt in",
        "z": "d716c1e95bfd1b2c",
        "name": "Gol MQTT",
        "topic": "futebol/gol",
        "qos": "0",
        "datatype": "json",
        "broker": "672549255251aba1",
        "inputs": 0,
        "x": 220,
        "y": 400,
        "wires": [
            [
                "8c94a50595a11f60"
            ]
        ]
    },
    {
        "id": "8c94a50595a11f60",
        "type": "ui_text",
        "z": "d716c1e95bfd1b2c",
        "group": "ui-group1",
        "order": 2,
        "width": 6,
        "height": 1,
        "name": "Gol Evento",
        "label": "Gol!",
        "format": "Time {{msg.payload.time}} marcou de {{msg.payload.distancia}}m",
        "layout": "row-spread",
        "x": 470,
        "y": 400,
        "wires": []
    },
    {
        "id": "9540ae5f6f98c386",
        "type": "mqtt in",
        "z": "d716c1e95bfd1b2c",
        "name": "MQTT Futebol",
        "topic": "futebol/+/telemetry",
        "qos": "0",
        "datatype": "json",
        "broker": "broker1",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 150,
        "y": 140,
        "wires": [
            [
                "31a48f819dfd28da"
            ]
        ]
    },
    {
        "id": "31a48f819dfd28da",
        "type": "function",
        "z": "d716c1e95bfd1b2c",
        "name": "Separar dados",
        "func": "let p = msg.payload;\n\nreturn [\n    { payload: p.heart_rate, topic: p.player + \"_heart\" },\n    { payload: p.speed, topic: p.player + \"_speed\" }\n];",
        "outputs": 2,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 370,
        "y": 140,
        "wires": [
            [
                "9375a7a77289d7fd"
            ],
            [
                "86e879d7a2f1c86a"
            ]
        ]
    },
    {
        "id": "86e879d7a2f1c86a",
        "type": "ui_chart",
        "z": "d716c1e95bfd1b2c",
        "name": "Velocidade",
        "group": "ui-group",
        "order": 1,
        "width": 0,
        "height": 0,
        "label": "Velocidade Jogador",
        "chartType": "line",
        "legend": "true",
        "xformat": "auto",
        "interpolate": "linear",
        "nodata": "",
        "dot": false,
        "ymin": "0",
        "ymax": "30",
        "removeOlder": 1,
        "removeOlderPoints": "",
        "removeOlderUnit": "3600",
        "cutout": 0,
        "useOneColor": false,
        "colors": [
            "#1f77b4",
            "#ff7f0e",
            "#2ca02c"
        ],
        "outputs": 1,
        "x": 620,
        "y": 200,
        "wires": [
            []
        ]
    },
    {
        "id": "9375a7a77289d7fd",
        "type": "ui_gauge",
        "z": "d716c1e95bfd1b2c",
        "name": "Batimento",
        "group": "ui-group1",
        "order": 1,
        "width": 0,
        "height": 0,
        "gtype": "gage",
        "title": "Batimento Cardíaco",
        "label": "bpm",
        "format": "Jogadora 1",
        "min": 50,
        "max": 200,
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "seg1": "",
        "seg2": "",
        "diff": false,
        "className": "",
        "x": 620,
        "y": 120,
        "wires": []
    },
    {
        "id": "e5c2597cec405823",
        "type": "global-config",
        "env": [],
        "modules": {
            "node-red-dashboard": "3.6.6"
        }
    }
]

## 9. Licença
Este projeto encontra‑se disponível sob a licença MIT (ou outra de sua preferência). Consulte o arquivo `LICENSE` caso deseje aplicar uma licença específica.

