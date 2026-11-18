# hangmanGame
jogo da forca p2p
módulo hangman.py que implementa a lógica da forca
=======
# HangmanGame Anotações — `hangman.py` e `game_host.py`

## 1. `hangman.py` — regras da forca

**Responsabilidade:** manter a palavra secreta, revelar letras, descontar vidas e identificar vitória ou derrota. Não controla rede nem turnos.

| Função / estrutura | O que faz |
|---|---|
| `normalize_word(word)` | Converte para minúsculas e remove acentos. |
| `validate_word(word)` | Verifica 3–20 caracteres e somente letras `a-z`. |
| `calculate_word_hash(word)` | Calcula o SHA-256 da palavra. |
| `GuessResult` | Resultado de uma tentativa: aceita, correta, motivo e quantidade revelada. |
| `HangmanGame.__init__()` | Inicializa palavra, vidas e letras tentadas. |
| `get_masked_word()` | Monta a palavra com letras descobertas e `_` nas posições ocultas. |
| `guess_letter(letter)` | Valida a letra, revela todas as ocorrências ou desconta uma vida. |
| `guess_word(word)` | Verifica o palpite da palavra inteira; acerto vence, erro custa uma vida. |
| `is_won()` / `is_lost()` | Verificam vitória / ausência de vidas. |
| `is_finished()` | Informa se houve vitória ou derrota. |
| `get_wrong_letters()` | Retorna as letras erradas. |
| `get_state()` | Retorna um dicionário com máscara, letras erradas, vidas e resultado. |

**Dados importantes:** `word`, `masked_word`, `remaining_lives`, `correct_letters`, `wrong_letters`, `used_letters`.

## 2. `game_host.py` - máquina de estados do host

O módulo `game_host.py` administra uma partida de forca em rede. O **host é autoritativo**: controla jogadores, turnos, palpites, pontuação, tempo e encerramento. A lógica da palavra fica em `HangmanGame`, importada de `hangman.py`.

## 2. Estruturas principais

- **`GameStatus`**: estados `IDLE` (sem partida), `OPEN` (aceita jogadores), `RUNNING` (em andamento) e `FINISHED` (encerrada).
- **`GameSession`**: guarda `game`, `game_id`, palavra/hash, jogadores, placares, vidas, turno (`current_turn`), sequência de estados (`seq`), timeout, resultado e desconectados.
- **`GameHost`**: coordena a sessão e a comunicação. Recebe `config`, `host_id` e `send_message` no construtor.

## 3. Funções por responsabilidade

### Configuração e comunicação

| Função | Essencial |
|---|---|
| `status` | Propriedade que retorna o estado da sessão ou `IDLE`. |
| `_config_value()` | Lê uma configuração, inclusive de `config['game']`, com valor padrão. |
| `_message()` | Monta mensagem com `type`, `msg_id`, `ttl`, `src` e `dst`. |
| `_send()` | Envia mensagem a um jogador usando a função de rede injetada. |
| `_broadcast()` | Envia a mesma categoria de mensagem a vários jogadores. |

### Criação, jogadores e turnos

| Função | Essencial |
|---|---|
| `create_game()` | Normaliza/valida palavra, configura limites e cria sessão `OPEN`. |
| `get_game_info()` | Retorna informações públicas, sem revelar a palavra. |
| `add_player()` | Inscreve jogador em `OPEN` e inicia placar em zero. |
| `get_current_player()` | Obtém o jogador da vez. |
| `next_turn()` | Avança circularmente: `(current_turn + 1) % len(players)`. |
| `remove_player()` / `_remove_player_locked()` | Remove jogador, ajusta turno e encerra se não restarem participantes. |
| `handle_player_disconnect()` | Trata queda de conexão e registra possibilidade de reconexão. |

### Temporização

| Função | Essencial |
|---|---|
| `_cancel_turn_timer()` | Cancela tarefa anterior e invalida timers antigos. |
| `_restart_turn_timer()` | Cria nova tarefa assíncrona de timeout. |
| `start_turn_timer()` | Aguarda `turn_timeout` com `asyncio.sleep()`. |
| `handle_turn_timeout()` / `_handle_turn_timeout_locked()` | Registra timeout, passa a vez e publica novo `STATE`. |

### Início, jogadas e atualização

| Função | Essencial |
|---|---|
| `start_game()` | Faz `OPEN → RUNNING`, envia `GAME_START`, primeiro `STATE` e inicia timer. |
| `broadcast_game_start()` | Comunica participantes, vidas, hash e duração do turno. |
| `validate_guess()` | Verifica estado, jogador da vez e validade de letra/palavra; retorna `None` se válido. |
| `process_guess()` | Valida, aplica palpite no `HangmanGame`, calcula pontos, responde e avança ou encerra. |
| `update_score()` | Soma pontos ao placar. |
| `check_game_over()` | Consulta vitória (`WON`) ou derrota (`LOST`). |
| `build_state()` | Monta estado público: palavra mascarada, erros, vidas, turno etc. |
| `broadcast_state()` | Incrementa `seq` e distribui `STATE`. |

### Encerramento

| Função | Essencial |
|---|---|
| `broadcast_game_over()` | Envia resultado, palavra, vencedor e placares. |
| `_finish_locked()` | Centraliza finalização: cancela timer, marca `FINISHED` e comunica fim. |
| `_finished_expiry()` | Após 60 s, limpa a sessão e retorna a `IDLE`. |
| `cancel_game()` | Aborta partida `OPEN` ou `RUNNING`. |
| `shutdown()` | Cancela partida/tarefas antes de fechar conexões. |

### Entrada de mensagens da rede

| Função | Mensagem tratada |
|---|---|
| `handle_game_list()` | `GAME_LIST` → `GAME_LIST_OK`. |
| `handle_join()` | `JOIN` → `JOIN_OK` ou `JOIN_ERR`; admite reconexão conhecida. |
| `handle_leave()` | `LEAVE` → `LEAVE_OK` ou `ERROR`. |
| `handle_guess()` | `GUESS` → encaminha para `process_guess()`. |
| `handle_message()` | Despachante: valida `ttl`/destino e escolhe o handler pelo `type`. |
