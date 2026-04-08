## 👩‍💻 Ground Truth: Repository Snapshot (Gabarito Oculto)

Note: Você NÃO TEM acesso à leitura dos arquivos reais na máquina do usuário. Portanto, a **TODO O REPOSITÓRIO E SEU CÓDIGO COMPLETO ENCONTRAM-SE AQUI ABAIXO**. Esta é sua única fonte da verdade para cobrar a arquitetura exata dele. Respeite todos os imports e módulos na mesma proporção deste gabarito.

### 1. `server/src/logger.py`
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

### 2. `server/src/config.py`
```python
import os
from dataclasses import dataclass

@dataclass(frozen=True)
class Config:
    """Configurações centralizadas e imutáveis do servidor."""
    PORT: int = int(os.environ.get("PORT", "8888"))
    LISTEN_ADDRESS: str = "0.0.0.0"
    STATIC_PATH: str = os.path.join(os.path.dirname(__file__), "..", "..", "client", "static")
    DEFAULT_PAGE: str = "index.html"
    LOG_LEVEL: str = "INFO"
    BOARD_SIZE: int = 3
    WIN_CONDITION: int = 3

config = Config()
```

### 3. `server/src/entities.py`
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

### 4. `server/src/logic.py`
```python
from src.entities import GameState

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
            player_o_id=self._state.player_o_id
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
        is_draw = all(cell is not None for row in board for cell in row)
        if is_draw:
            return "Draw", True
        return None, False

    def assign_player(self, player_id: str) -> str | None:
        if self._state.player_x_id is None:
            self._state = GameState(
                board=self._state.board, current_turn=self._state.current_turn,
                winner=self._state.winner, game_over=self._state.game_over,
                player_x_id=player_id, player_o_id=self._state.player_o_id
            )
            return "X"
        elif self._state.player_o_id is None:
            self._state = GameState(
                board=self._state.board, current_turn=self._state.current_turn,
                winner=self._state.winner, game_over=self._state.game_over,
                player_x_id=self._state.player_x_id, player_o_id=player_id
            )
            return "O"
        return None

    def remove_player(self, player_id: str) -> None:
        player_x = self._state.player_x_id if self._state.player_x_id != player_id else None
        player_o = self._state.player_o_id if self._state.player_o_id != player_id else None
        self._state = GameState(
            board=self._state.board, current_turn=self._state.current_turn,
            winner=self._state.winner, game_over=self._state.game_over,
            player_x_id=player_x, player_o_id=player_o
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
            current_turn=new_turn
        )
```

### 5. `server/src/room_manager.py`
```python
import uuid
from src.logic import GameLogic

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

### 6. `server/src/handlers.py`
```python
import json
from tornado.web import RequestHandler
from tornado.websocket import WebSocketHandler
from src.room_manager import RoomManager
from src.logger import get_logger

logger = get_logger("Handlers")
room_manager = RoomManager()

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

        if hasattr(self, 'game') and self.game:
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
            if hasattr(client, 'room_id') and client.room_id == self.room_id:
                client.write_message(message)
```

### 7. `server/main.py`
```python
import socket
from tornado.ioloop import IOLoop
from tornado.web import Application, StaticFileHandler
from src.handlers import CreateRoomHandler, GameWebSocket
from src.logger import setup_logger, get_logger
from src.config import config

setup_logger()
logger = get_logger("ServidorTornado")

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
            {
                "path": config.STATIC_PATH,
                "default_filename": config.DEFAULT_PAGE
            }
        ),
    ]
    return Application(url_routes, debug=True)

if __name__ == "__main__":
    app = make_app()
    port = config.PORT
    ip = get_local_ip()

    logger.info("=" * 50)
    logger.info("⚔️  Mestre Jedi - Jogo da Velha (Tornado Server)")
    logger.info(f"🌍 IP Local (LAN) Detectado: {ip}")
    logger.info(f"🔗 Porta de Escuta: {port}")
    logger.info(f"🚀 Link Base do Servidor: http://{ip}:{port}")
    logger.info("=" * 50)
    logger.info("Aguardando Conexões e Criação de Salas...")

    app.listen(port, address=config.LISTEN_ADDRESS)
    IOLoop.current().start()
```

### 8. `client/static/ui.js`
*(Omitido sem alterações de lógica para concisão do gabarito, usar o original se necessário)*

### 9. `client/static/ws.js`
*(Omitido sem alterações de lógica para concisão do gabarito, usar o original se necessário)*

### 10. `client/static/main.js`
*(Omitido sem alterações de lógica para concisão do gabarito, usar o original se necessário)*

### 11. `client/static/index.html`
```html
<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mestre Jedi - Jogo da Velha LAN</title>
    <link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 100 100%22><text y=%22.9em%22 font-size=%2290%22>⚔️</text></svg>">
    <meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
    <meta http-equiv="Pragma" content="no-cache">
    <meta http-equiv="Expires" content="0">
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="/style.css">
</head>
<body class="bg-slate-950 text-slate-50 flex items-center justify-center min-h-screen p-6 antialiased">
    <div class="bg-slate-900/80 backdrop-blur-lg rounded-2xl shadow-2xl text-center border border-slate-800 w-full max-w-xs sm:max-w-sm md:max-w-md ring-1 ring-white/5 p-6 sm:p-8 flex flex-col gap-4">
        <h1 class="text-3xl sm:text-4xl font-extrabold tracking-tight bg-gradient-to-r from-sky-400 to-indigo-400 bg-clip-text text-transparent">Jogo da Velha</h1>
        <div id="status-display" class="font-medium text-sm text-slate-400">Inicializando conexão...</div>

        <div id="setup-panel" class="flex flex-col gap-3">
            <p class="text-sm md:text-base text-slate-400 text-balance leading-relaxed mx-auto">Inicie uma sala e convide um adversário pelo link ou QR Code.</p>
            <button id="create-room-btn" class="w-full py-3 bg-sky-500 hover:bg-sky-400 text-slate-950 font-bold rounded-xl shadow-lg transition-all active:scale-95">⚔️ Criar Nova Partida</button>
        </div>

        <div id="invite-panel" class="hidden flex-col items-center gap-4 p-4 bg-slate-950/60 rounded-xl border border-slate-800">
            <p class="text-xs text-slate-400 font-medium uppercase tracking-widest">Convite para a Sala</p>
            <img id="qrcode-img" src="" alt="QR Code Convite" class="hidden w-36 h-36 sm:w-48 sm:h-48 md:w-56 md:h-56 rounded-xl bg-white p-1.5 shadow-lg" />
            <a href="#" id="invite-link" class="text-sky-400 font-semibold text-xs underline truncate w-full block" target="_blank"></a>
        </div>

        <div id="board" class="hidden grid-cols-3 gap-3 mx-auto w-fit transition-all duration-300">
            <!-- 9 divs de tabuleiro -->
            <div class="cell w-20 sm:w-24 md:w-28 aspect-square bg-slate-800 flex items-center justify-center text-4xl font-bold rounded-xl border border-slate-700/50 shadow-sm transition-all hover:bg-slate-700 select-none" data-row="0" data-col="0"></div>
            <div class="cell w-20 sm:w-24 md:w-28 aspect-square bg-slate-800 flex items-center justify-center text-4xl font-bold rounded-xl border border-slate-700/50 shadow-sm transition-all hover:bg-slate-700 select-none" data-row="0" data-col="1"></div>
            <div class="cell w-20 sm:w-24 md:w-28 aspect-square bg-slate-800 flex items-center justify-center text-4xl font-bold rounded-xl border border-slate-700/50 shadow-sm transition-all hover:bg-slate-700 select-none" data-row="0" data-col="2"></div>
            <div class="cell w-20 sm:w-24 md:w-28 aspect-square bg-slate-800 flex items-center justify-center text-4xl font-bold rounded-xl border border-slate-700/50 shadow-sm transition-all hover:bg-slate-700 select-none" data-row="1" data-col="0"></div>
            <div class="cell w-20 sm:w-24 md:w-28 aspect-square bg-slate-800 flex items-center justify-center text-4xl font-bold rounded-xl border border-slate-700/50 shadow-sm transition-all hover:bg-slate-700 select-none" data-row="1" data-col="1"></div>
            <div class="cell w-20 sm:w-24 md:w-28 aspect-square bg-slate-800 flex items-center justify-center text-4xl font-bold rounded-xl border border-slate-700/50 shadow-sm transition-all hover:bg-slate-700 select-none" data-row="1" data-col="2"></div>
            <div class="cell w-20 sm:w-24 md:w-28 aspect-square bg-slate-800 flex items-center justify-center text-4xl font-bold rounded-xl border border-slate-700/50 shadow-sm transition-all hover:bg-slate-700 select-none" data-row="2" data-col="0"></div>
            <div class="cell w-20 sm:w-24 md:w-28 aspect-square bg-slate-800 flex items-center justify-center text-4xl font-bold rounded-xl border border-slate-700/50 shadow-sm transition-all hover:bg-slate-700 select-none" data-row="2" data-col="1"></div>
            <div class="cell w-20 sm:w-24 md:w-28 aspect-square bg-slate-800 flex items-center justify-center text-4xl font-bold rounded-xl border border-slate-700/50 shadow-sm transition-all hover:bg-slate-700 select-none" data-row="2" data-col="2"></div>
        </div>

        <button id="reset-btn" class="hidden w-full py-3 bg-slate-100 hover:bg-white text-slate-900 font-bold rounded-xl shadow-lg transition-all active:scale-95">🔄 Jogar Novamente</button>
    </div>
    <script type="module" src="/main.js?v=6"></script>
</body>
</html>
```

### 12. `client/static/style.css`
```css
/* Mestre Jedi Design System - Jogo da Velha */
@import url('https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;600;800&display=swap');

:root {
    --glass-bg: rgba(15, 23, 42, 0.8);
    --glass-border: rgba(255, 255, 255, 0.08);
}

body {
    font-family: 'Outfit', sans-serif;
    background: radial-gradient(circle at top left, #0f172a, #020617);
}

.glass {
    background: var(--glass-bg);
    backdrop-filter: blur(12px);
    -webkit-backdrop-filter: blur(12px);
    border: 1px solid var(--glass-border);
}

.cell {
    transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
}

.cell:active {
    transform: scale(0.92);
}

::-webkit-scrollbar { width: 8px; }
::-webkit-scrollbar-track { background: #020617; }
::-webkit-scrollbar-thumb { background: #1e293b; border-radius: 10px; }
::-webkit-scrollbar-thumb:hover { background: #334155; }
```

### 13. `Makefile`
```makefile
.PHONY: run tunnel dev stop help

# Inicia o servidor Tornado
run:
	python3 server/main.py

# Abre tunel reverso via localhost.run
tunnel:
	killall ssh 2>/dev/null; ssh -R 80:localhost:8888 nokey@localhost.run -o StrictHostKeyChecking=no

# Inicia servidor e tunel em paralelo
dev:
	$(MAKE) run &
	$(MAKE) tunnel

# Encerra processos do servidor e túneis SSH
stop:
	@lsof -ti :8888 | xargs kill -9 2>/dev/null && echo "Servidor encerrado." || echo "Nenhum servidor rodando."
	@killall ssh 2>/dev/null && echo "Tunel SSH encerrado." || echo "Nenhum tunel ativo."

# Lista os comandos disponíveis
help:
	@echo ""
	@echo "  Comandos:"
	@echo "  make run     - Inicia servidor"
	@echo "  make tunnel  - Abre tunel SSH"
	@echo "  make dev     - Servidor + Tunel"
	@echo "  make stop    - Encerra tudo"
	@echo ""
```
