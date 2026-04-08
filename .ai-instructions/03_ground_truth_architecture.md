## 🏛️ Ground Truth: Arquitetura e Fluxo do Jogo (Gabarito Oculto do Mestre)

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
    S->>S: Lógica Valida Jogada (Entities)
    S-->>P1: Broadcast: {"type": "update", "state": {...}}
    S-->>P2: Broadcast: {"type": "update", "state": {...}}
```

### Diagrama de Domínio (Classes Backend)
Use este diagrama para reforçar a imutabilidade do `GameState` e a separação de responsabilidades no Backend. O Padawan não pode misturar lógica na controller de Websocket!

```mermaid
classDiagram
    class Config {
        <<Frozen DataClass>>
        +int PORT
        +str LISTEN_ADDRESS
        +str STATIC_PATH
        +str DEFAULT_PAGE
    }
    class RoomManager {
        +dict rooms
        +create_room() string
        +get_room(id) GameLogic
        +delete_room(id)
    }
    class GameLogic {
        -GameState _state
        +state
        +make_move(row, col, symbol)
        +assign_player(id)
        +can_start()
        +reset()
    }
    class GameState {
        <<Immutable DataClass>>
        +list board
        +str current_turn
        +str winner
        +bool game_over
        +to_dict() dict
    }
    RoomManager "1" *-- "many" GameLogic : gerencia salas
    GameLogic "1" *-- "1" GameState : recria o estado
    RoomManager ..> Config : depende de config (Singleton)
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

