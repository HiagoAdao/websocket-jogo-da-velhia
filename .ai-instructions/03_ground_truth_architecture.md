## 🏛️ Ground Truth: Arquitetura e Fluxo do Jogo (Gabarito Oculto do Mestre)

### Estrutura de Camadas (Clean Architecture)

```
jogo-da-velha-websocket/    ← Raiz do Projeto
├── main.py                 ← Bootstrap: wiring de todas as camadas
├── Makefile
├── requirements.txt
│
├── core/                   ← Camada de Infraestrutura/Configuração
│   ├── config.py           ← Configurações centralizadas (LCD, Buzzer, Rede)
│   └── logger.py           ← Logger centralizado
│
├── game/                   ← Camada de Domínio (Pura)
│   ├── entities.py         ← GameState (modelo imutável)
│   └── logic.py            ← GameLogic (regras)
│
├── hardware/               ← Camada de Acesso a Hardware (HIL)
│   ├── display.py          ← LCDManager (RPLCD)
│   ├── buzzer.py           ← BuzzerManager (PWM)
│   └── system.py           ← HardwareSystem (Fachada/Facade)
│
├── server/                 ← Camada de Aplicação (Tornado)
│   ├── handlers.py         ← WebSocket + HTTP
│   └── manager.py          ← Orquestração de salas
│
└── client/                 ← Camada de Apresentação (Frontend)
    └── static/
        ├── index.html
        └── [js modules]

**Regra de Dependência (Dependency Rule):**
- `game/`: Totalmente isolada (Domínio).
- `core/`: Transversal, usada por todos.
- `hardware/`: Depende de `core/`.
- `server/`: Depende de `game/`, `core/` e `hardware/`.
- `main.py`: Faz o wiring de tudo.

---

### Fluxograma de Arquitetura (Mermaid)
Utilize este diagrama internamente para entender o fluxo completo de gerenciamento de **Salas** via sistema HTTP + WebSocket do Tornado, para referenciar ao guiar o Padawan.

```mermaid
sequenceDiagram
    participant P1 as 🧑💻 Jogador 1 (Criador)
    participant P2 as 🧑💻 Jogador 2 (Convidado)
    participant S as 🚀 Servidor Tornado

    P1->>S: GET /api/create-room
    S-->>P1: { "room_id": "abc12", "link": "http://IP/?sala=abc12" }
    
    P1->>S: ws://IP/ws?sala=abc12 (Abre Conexão)
    S-->>P1: {"type": "init", "symbol": "X", "room": "abc12"}
    S-->>P1: {"type": "wait", "message": "Aguardando..."}
    
    P2->>S: ws://IP/ws?sala=abc12 (Abre Conexão por Link)
    S-->>P2: {"type": "init", "symbol": "O", "room": "abc12"}
    
    S-->>P1: {"type": "update", "state": {...}} (O Jogo Começa)
    S-->>P2: {"type": "update", "state": {...}} (O Jogo Começa)

    Note over P1,S: Turno do Jogador X
    P1->>S: {"action": "move", "row": 0, "col": 0}
    S->>S: Lógica Valida Jogada (game/logic.py)
    S-->>P1: Broadcast: {"type": "update", "state": {...}}
    S-->>P2: Broadcast: {"type": "update", "state": {...}}
```

### Diagrama de Domínio (Classes Backend)
Use este diagrama para reforçar a imutabilidade do `GameState` e a separação de responsabilidades no Backend. O Padawan não pode misturar lógica na controller de WebSocket!

```mermaid
classDiagram
    class Config {
        <<core/config.py>>
        +PORT
        +LCD: LCDSettings
        +BUZZER: BuzzerSettings
    }
    class HardwareSystem {
        <<hardware/system.py>>
        -lcd: LCDManager
        -buzzer: BuzzerManager
        +notify_victory(winner)
        +update_game_status(...)
    }
    class LCDManager {
        <<hardware/display.py>>
        +show_status(...)
        +show_idle(ip)
    }
    class BuzzerManager {
        <<hardware/buzzer.py>>
        +play_mario_victory()
        +beep(...)
    }
    class RoomManager {
        <<server/manager.py>>
        +rooms: dict
        +create_room()
    }
    HardwareSystem "1" *-- "1" LCDManager
    HardwareSystem "1" *-- "1" BuzzerManager
    RoomManager "1" *-- "many" GameLogic
    GameLogic "1" *-- "1" GameState
```

### Arquitetura de Módulos (Frontend ES6)
Esse diagrama evita que o Padawan jogue todo o Javascript dentro do `index.html`. Cobre dele os imports isolados.

```mermaid
graph TD
    HTML["index.html (Tailwind CDN)"] -->|"type module"| Main["main.js (Controllers / Eventos)"]
    Main -->|"import"| UI["ui.js (Gerência de DOM)"]
    Main -->|"import"| WS["ws.js (Conexão Localhost)"]
    WS -->|"importa helper gráfico"| UI
```
