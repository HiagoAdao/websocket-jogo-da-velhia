# 07 — Integração de Hardware: Raspberry Pi, LCD e Buzzer

## 🎓 Diretriz Pedagógica

Mestre Jedi, o hardware é a "casca" física do sistema. Quando o Padawan avançar para esta etapa, você deve guiá-lo na montagem física e na compreensão de como o software se comunica com o mundo real via GPIO (General Purpose Input/Output).

---

## 🛠️ Tutorial de Hardware

### 1. Preparação da Raspberry Pi
Mestre Jedi, antes de instruir qualquer instalação, peça ao Padawan para verificar se as bibliotecas de GPIO já não estão presentes na **imagem oficial** do Raspberry Pi OS.
- **Como verificar**: `python3 -c "import RPi.GPIO; print('GPIO OK')"`
- Caso não estejam presentes, oriente a instalação:
```bash
sudo apt update
sudo apt install python3-rpi.gpio
```

### 2. Ligação do Display LCD (16x2)
O display utiliza uma interface de 4 bits. Oriente o Padawan a seguir a imagem `rasp-lcd.png` e as conexões abaixo (Padrão BCM):

| LCD Pin | Nome | Destino na Raspberry Pi (GPIO / Pin) |
|---------|------|--------------------------------------|
| 1       | VSS  | GND (Pin 6 ou similar)               |
| 2       | VDD  | 5V (Pin 2)                           |
| 3       | V0   | Terminal Central do Potenciômetro    |
| 4       | RS   | **GPIO 18** (Pin 12)                 |
| 5       | RW   | GND (Pin 6 ou similar)               |
| 6       | E    | **GPIO 23** (Pin 16)                 |
| 11      | D4   | **GPIO 12** (Pin 32)                 |
| 12      | D5   | **GPIO 16** (Pin 36)                 |
| 13      | D6   | **GPIO 20** (Pin 38)                 |
| 14      | D7   | **GPIO 21** (Pin 40)                 |
| 15      | A    | 5V via Resistor 220Ω (ou Direto)     |
| 16      | K    | GND                                  |

> O potenciômetro é crucial para ajustar o contraste. Se o Padawan relatar que "não vê nada", mande-o girar o potenciômetro!

### 3. Ligação do Buzzer
O Buzzer é mais simples e não precisa de imagem. Ele deve ser conectado em uma porta PWM para o tema do Mario:
- **Polo Positivo (+)**: Conectar no **GPIO 14** (Pin 8).
- **Polo Negativo (-)**: Conectar em qualquer pino **GND** (ex: Pin 9 ou 14).

---

## 🔍 Visão de Código (A Camada de Hardware)

Domine estes conceitos para explicar ao Padawan:

### O Facade (`hardware/system.py`)
O sistema utiliza o padrão **Facade** para que o servidor Tornado não precise falar diretamente com o LCD ou Buzzer. O `HardwareSystem` orquestra ambos.

### Temas Sonoros (`hardware/buzzer.py`)
- `play_imperial_march()`: Toca ao iniciar o servidor (Boot).
- `play_mario_victory()`: Toca quando alguém vence a partida (The Victory Theme!).

### Fallback (HAS_HARDWARE)
O código é resiliente. Se a `RPLCD` ou `RPi.GPIO` não estiverem instaladas (ex: rodando no PC do Padawan), o sistema entrará em **Modo Simulação**, logando as ações no terminal em vez de falhar.

---

## 🧠 Desafio de Hardware

Pergunte ao Padawan antes dele rodar:
- *"Por que usamos GPIO no modo BCM e não no modo BOARD nas nossas configurações?"*
- *"O que aconteceria se tentássemos acessar o hardware sem privilégios de ROOT (sudo) em algumas versões de OS?"*
- *"Como o software sabe se o LCD está realmente conectado ou se está apenas 'falando para o vazio'?"* (Resposta: O protocolo do LCD 16x2 é de mão única, o software não sabe, a menos que usemos I2C ou verifiquemos resistividade).
