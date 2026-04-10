## 👩‍💻 Ground Truth: Repository Snapshot (Gabarito Oculto)

Note: Você NÃO TEM acesso à leitura dos arquivos reais na máquina do usuário. Portanto, **TODO O REPOSITÓRIO E SEU CÓDIGO COMPLETO ENCONTRAM-SE AQUI ABAIXO**. Esta é sua única fonte da verdade para cobrar a arquitetura exata dele. Respeite todos os imports e módulos na mesma proporção deste gabarito.

A arquitetura segue **Clean Architecture** com separação explícita de camadas:
- **Raiz**: `config.py` e `logger.py` são infraestrutura transversal
- **`game/`**: Domínio puro (zero dependência de framework)
- **`server/`**: Camada de aplicação (Tornado-specific)

---

### 1. `core/logger.py`
```python
import logging

def setup_logger() -> None:
    logging.basicConfig(
        level=logging.INFO,
        format="[%(asctime)s] [%(levelname)s] [%(name)s] %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )

def get_logger(name: str) -> logging.Logger:
    return logging.getLogger(name)
```

### 2. `core/config.py`
```python
import os
from dataclasses import dataclass, field
from typing import List

@dataclass(frozen=True)
class BuzzerSettings:
    PIN: int = 14
    DEFAULT_FREQUENCY: int = 2500

@dataclass(frozen=True)
class LCDSettings:
    RS: int = 18
    EN: int = 23
    DATA_PINS: List[int] = field(default_factory=lambda: [12, 16, 20, 21])
    COLS: int = 16
    ROWS: int = 2
    SSH_MESSAGE: str = "Acesso via SSH"

@dataclass(frozen=True)
class Config:
    PORT: int = int(os.environ.get("PORT", "8888"))
    LISTEN_ADDRESS: str = "0.0.0.0"
    STATIC_PATH: str = os.path.join(os.path.dirname(__file__), "..", "client", "static")
    # ... (outras configs)
    LCD: LCDSettings = field(default_factory=LCDSettings)
    BUZZER: BuzzerSettings = field(default_factory=BuzzerSettings)

config = Config()
```

### 3. `game/entities.py` ← **camada de domínio**
```python
from dataclasses import dataclass, field
from typing import Optional, Any

@dataclass(frozen=True)
class GameState:
    """Representação imutável do estado de uma partida de Jogo da Velha."""
    board: list[list[Optional[str]]] = field(
        default_factory=lambda: [[None, None, None] for _ in range(3)]
    )
    current_turn: str = "X"
    winner: Optional[str] = None
    game_over: bool = False
    player_x_id: Optional[str] = None
    player_o_id: Optional[str] = None

    def to_dict(self) -> dict[str, Any]:
        """Converte o estado para um dicionário serializável."""
        return {
            "board": self.board,
            "current_turn": self.current_turn,
            "winner": self.winner,
            "game_over": self.game_over,
            "player_x": {"symbol": "X", "active": self.player_x_id is not None},
            "player_o": {"symbol": "O", "active": self.player_o_id is not None},
        }
```

### 4. `game/logic.py` ← **camada de domínio**
```python
from game.entities import GameState

class GameLogic:
    def __init__(self) -> None:
        self._state = GameState()

    @property
    def state(self) -> GameState:
        return self._state

    def make_move(self, row: int, col: int, symbol: str) -> bool:
        if self._state.game_over:
            return False
        if not (0 <= row < 3 and 0 <= col < 3):
            return False
        if self._state.board[row][col] is not None:
            return False
        if symbol != self._state.current_turn:
            return False

        new_board = [list(r) for r in self._state.board]
        new_board[row][col] = symbol

        winner, game_over = self._check_status(new_board)

        next_turn = "O" if self._state.current_turn == "X" else "X"
        if game_over:
            next_turn = self._state.current_turn

        self._state = GameState(
            board=new_board,
            current_turn=next_turn,
            winner=winner,
            game_over=game_over,
            player_x_id=self._state.player_x_id,
            player_o_id=self._state.player_o_id,
        )
        return True

    def _check_status(self, board: list[list[str | None]]) -> tuple[str | None, bool]:
        for i in range(3):
            if board[i][0] == board[i][1] == board[i][2] and board[i][0] is not None:
                return board[i][0], True
            if board[0][i] == board[1][i] == board[2][i] and board[0][i] is not None:
                return board[0][i], True
        if board[0][0] == board[1][1] == board[2][2] and board[0][0] is not None:
            return board[0][0], True
        if board[0][2] == board[1][1] == board[2][0] and board[0][2] is not None:
            return board[0][2], True
        if all(cell is not None for row in board for cell in row):
            return "Draw", True
        return None, False

    def assign_player(self, player_id: str) -> str | None:
        if self._state.player_x_id is None:
            self._state = GameState(
                board=self._state.board, current_turn=self._state.current_turn,
                winner=self._state.winner, game_over=self._state.game_over,
                player_x_id=player_id, player_o_id=self._state.player_o_id,
            )
            return "X"
        elif self._state.player_o_id is None:
            self._state = GameState(
                board=self._state.board, current_turn=self._state.current_turn,
                winner=self._state.winner, game_over=self._state.game_over,
                player_x_id=self._state.player_x_id, player_o_id=player_id,
            )
            return "O"
        return None

    def remove_player(self, player_id: str) -> None:
        player_x = self._state.player_x_id if self._state.player_x_id != player_id else None
        player_o = self._state.player_o_id if self._state.player_o_id != player_id else None
        self._state = GameState(
            board=self._state.board, current_turn=self._state.current_turn,
            winner=self._state.winner, game_over=self._state.game_over,
            player_x_id=player_x, player_o_id=player_o,
        )

    def is_full(self) -> bool:
        return self._state.player_x_id is not None and self._state.player_o_id is not None

    def can_start(self) -> bool:
        return self.is_full()

    def reset(self) -> None:
        new_turn = self._state.winner if self._state.winner in ["X", "O"] else "X"
        self._state = GameState(
            player_x_id=self._state.player_x_id,
            player_o_id=self._state.player_o_id,
            current_turn=new_turn,
        )
```

### 5. `server/manager.py` ← **camada de aplicação**
```python
import uuid
from game.logic import GameLogic

class RoomManager:
    def __init__(self) -> None:
        self.rooms: dict[str, GameLogic] = {}

    def create_room(self) -> str:
        room_id = str(uuid.uuid4())[:8]
        self.rooms[room_id] = GameLogic()
        return room_id

    def get_room(self, room_id: str) -> GameLogic | None:
        return self.rooms.get(room_id)

    def delete_room(self, room_id: str) -> None:
        if room_id in self.rooms:
            del self.rooms[room_id]
```

### 6. `server/handlers.py`
```python
import json
from tornado.web import RequestHandler
from tornado.websocket import WebSocketHandler
from core.logger import get_logger

logger = get_logger("Handlers")

# ... (restante do código)
```

### 6.5. `hardware/buzzer.py`
```python
# ... (implementação com play_mario_victory)
```

class CreateRoomHandler(RequestHandler):
    def get(self) -> None:
        room_id = room_manager.create_room()
        host = self.request.host
        link = f"http://{host}/?sala={room_id}"
        logger.info(f"Nova sala criada: {room_id} | Acesso via: {link}")
        self.write({"room_id": room_id, "link": link})

class GameWebSocket(WebSocketHandler):
    _clients: list["GameWebSocket"] = []

    def check_origin(self, origin: str) -> bool:
        return True

    def open(self) -> None:
        GameWebSocket._clients.append(self)
        self.room_id = self.get_argument("sala", None)
        self.player_id = id(self)
        self.symbol: str | None = None

        if not self.room_id:
            self._send_error("ID da Sala não fornecido. Use ?sala=ID")
            return

        self.game = room_manager.get_room(self.room_id)
        if not self.game:
            self._send_error("Sala não encontrada!")
            return

        if self.game.is_full():
            self._send_error("A sala já está cheia.")
            return

        self.symbol = self.game.assign_player(self.player_id)
        self.write_message(json.dumps({"type": "init", "symbol": self.symbol, "room": self.room_id}))

        if self.game.can_start():
            self._broadcast_state()
        else:
            self.write_message(json.dumps({"type": "wait", "message": "Aguardando o segundo jogador entrar pelo link..."}))

    def on_message(self, message: str | bytes) -> None:
        if not self.game or not self.game.can_start():
            self._send_error("O jogo ainda não começou!")
            return

        try:
            data = json.loads(message)
            if not isinstance(data, dict):
                return
            action = data.get("action")
            if action == "move":
                row = data.get("row")
                col = data.get("col")
                if isinstance(row, int) and isinstance(col, int) and self.symbol:
                    if self.game.make_move(row, col, self.symbol):
                        self._broadcast_state()
            elif action == "reset":
                self.game.reset()
                self._broadcast_state()
        except (json.JSONDecodeError, TypeError):
            logger.error("Falha ao processar mensagem JSON inválida")

    def on_close(self) -> None:
        if self in GameWebSocket._clients:
            GameWebSocket._clients.remove(self)
        if hasattr(self, "game") and self.game:
            self.game.remove_player(self.player_id)
            if not self.game.is_full():
                self._broadcast_wait("O oponente desconectou. Aguardando reconexão...")

    def _send_error(self, message: str) -> None:
        self.write_message(json.dumps({"type": "error", "message": message}))
        self.close()

    def _broadcast_state(self) -> None:
        payload = json.dumps({"type": "update", "state": self.game.state.to_dict()})
        self._send_to_room(payload)

    def _broadcast_wait(self, message: str) -> None:
        payload = json.dumps({"type": "wait", "message": message})
        self._send_to_room(payload)

    def _send_to_room(self, message: str) -> None:
        for client in GameWebSocket._clients:
            if hasattr(client, "room_id") and client.room_id == self.room_id:
                client.write_message(message)
```

### 7. `main.py`
```python
import socket
from tornado.ioloop import IOLoop
from tornado.web import Application, StaticFileHandler
from server.handlers import CreateRoomHandler, GameWebSocket
from core.logger import setup_logger, get_logger
from core.config import config
from hardware.system import HardwareSystem

setup_logger()
logger = get_logger("ServidorTornado")
hardware = HardwareSystem()

def get_local_ip() -> str:
    socket_connection = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    try:
        socket_connection.connect(("8.8.8.8", 1))
        return socket_connection.getsockname()[0]
    except Exception:
        return "127.0.0.1"
    finally:
        socket_connection.close()

def make_app() -> Application:
    url_routes = [
        (r"/api/create-room", CreateRoomHandler),
        (r"/ws", GameWebSocket),
        (
            r"/(.*)",
            StaticFileHandler,
            {"path": config.STATIC_PATH, "default_filename": config.DEFAULT_PAGE},
        ),
    ]
    return Application(url_routes, debug=True)

if __name__ == "__main__":
    app = make_app()
    ip = get_local_ip()
    
    # Inicializa hardware
    hardware.startup_sequence(ip)
    
    app.listen(config.PORT, address=config.LISTEN_ADDRESS)
    IOLoop.current().start()
```

### 8. `Makefile`
```makefile
.PHONY: setup run tunnel tunnel-lhr tunnel-serveo dev dev-lhr dev-serveo stop help

# Alvo para configurar o ambiente (Criar venv e instalar dependências)
setup:
	python3 -m venv venv
	./venv/bin/pip install -r requirements.txt

# Inicia o servidor usando a venv
run:
	./venv/bin/python3 main.py

# Tunel via localhost.run (URL aleatória gerada automaticamente)
tunnel-lhr:
	ssh -R 80:localhost:8888 nokey@localhost.run -o StrictHostKeyChecking=no

# Tunel via serveo.net (subdomínio fixo: https://velhia.serveo.net)
tunnel-serveo:
	ssh -R velhia:80:localhost:8888 serveo.net -o StrictHostKeyChecking=no

# Atalho padrão → serveo (subdomínio fixo)
tunnel: tunnel-serveo

# Servidor + serveo.net (padrão)
dev: dev-serveo

# Servidor + localhost.run
dev-lhr:
	$(MAKE) run &
	$(MAKE) tunnel-lhr

# Servidor + serveo.net
dev-serveo:
	$(MAKE) run &
	$(MAKE) tunnel-serveo

# Encerra processos do servidor e túneis
stop:
	@lsof -ti :8888 | xargs kill -9 2>/dev/null && echo "Servidor encerrado." || echo "Nenhum servidor rodando."
	@pkill -f "ssh.*serveo.net" 2>/dev/null && echo "Túnel serveo.net encerrado." || echo "Nenhum túnel serveo.net ativo."
	@pkill -f "ssh.*localhost.run" 2>/dev/null && echo "Túnel localhost.run encerrado." || echo "Nenhum túnel localhost.run ativo."

# Lista os comandos disponíveis
help:
	@echo ""
	@echo "  Comandos:"
	@echo "  make setup         - Configura ambiente virtual"
	@echo "  make run           - Inicia servidor"
	@echo "  make tunnel-lhr    - Tunel via localhost.run (URL aleatória)"
	@echo "  make tunnel-serveo - Tunel via serveo.net (https://velhia.serveo.net)"
	@echo "  make tunnel        - Atalho para tunnel-serveo"
	@echo "  make dev-lhr       - Servidor + localhost.run"
	@echo "  make dev-serveo    - Servidor + serveo.net"
	@echo "  make dev           - Atalho para dev-serveo"
	@echo "  make stop          - Encerra tudo"
	@echo ""
```
