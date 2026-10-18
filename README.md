# A Ponte: um agente A2A com MCP por Dentro

Onde termina a profundidade e começa o alcance

Projeto: MBA Engenharia de Software com IA, curso de MCP e A2A Fase do projeto: Desafio inicial

Conclusão do desafio MBA A2A-mcp. Neste projeto contém dois processos independentes: o
servidor MCP em `servidor-mcp/` e o agente A2A em `agente/`. O agente fala com
o MCP exclusivamente por HTTP/JSON-RPC; independentee das funções de domínio.

## Como Executar

Requisitos: Python 3.10 ou mais recente. A implementação usa somente a biblioteca padrão, portanto não
há dependências de pacote para instalar.

Em Linux/macOS, a partir da raiz do clone:

```bash
export REQUEST_STATE_SECRET="$(python3 -c 'import secrets; print(secrets.token_hex(32))')"
python3 servidor-mcp/server.py
```

Em outro terminal, ainda na raiz do clone:

```bash
python3 agente/server.py
```

No PowerShell:

```powershell
$env:REQUEST_STATE_SECRET = python -c "import secrets; print(secrets.token_hex(32))"
python servidor-mcp/server.py
```

Em outro terminal, na raiz do clone, suba o agente:

```powershell
python agente/server.py
```

O segredo é necessário somente no MCP. Para retomar um `requestState` após
um reinício do MCP, preserve o mesmo segredo no processo reiniciado.

O servidor MCP escuta `http://127.0.0.1:7301/mcp`; o agente A2A escuta
`http://127.0.0.1:7300`, com card em
`/.well-known/agent-card.json` e JSON-RPC em `/a2a`.

Com os dois processos recém-iniciados, rode:

```bash
python3 validador/validar.py --agente http://localhost:7300 --mcp http://localhost:7301
```

No Windows, o equivalente é:

```powershell
python validador/validar.py --agente http://localhost:7300 --mcp http://localhost:7301
```

O segredo deve ter pelo menos 32 bytes aleatórios. Nunca coloque o valor usado
no repositório.

## Onde a ponte acontece

Em `agente/server.py`, `MCPClient._call()` recebe a resposta HTTP crua e
`_complete_from_mcp()` reconhece `resultType=input_required`. Nesse ponto o
agente não responde à elicitation: guarda o `requestState` como string opaca,
a chave e o enum de alternativas na Task, muda o estado para
`TASK_STATE_INPUT_REQUIRED` e publica somente `alternativas: ...`. Em uma nova
mensagem `escolha=...`, `_process_continuation()` usa o token guardado,
mantém a chave original e envia um novo `tools/call` com `inputResponses`.
O servidor em `servidor-mcp/server.py` valida e abre o token, reconstrói os
argumentos selados, aplica `accept`, `decline` ou `cancel`, e retorna
`complete`. O agente transforma o resultado em artifact A2A sem expor o token.

## Decisões técnicas

- O retry de `reservar_sala` com `inputResponses` é tratado em um ramo
  exclusivo, antes do fluxo normal de reserva. Ele usa somente os argumentos
  do `requestState` validado e a sala escolhida entre as alternativas seladas.
  Argumentos crus adulterados não criam reservas adicionais; recusa,
  cancelamento e estado inválido não gravam reservas.

- Python 3.10+ e biblioteca padrão para tornar o clone reproduzível sem
  dependência implícita. O transporte implementa diretamente o wire contract
  JSON-RPC/Streamable HTTP cobrado pelo validador, incluindo
  `MCP-Protocol-Version`, `Mcp-Method`, `Mcp-Name`, `_meta`, `tools/list`,
  `resources/read` e `input_required`.
- `requestState` é um envelope versionado `v1.<payload>.<assinatura>` com
  payload Base64URL, HMAC-SHA256 e chave exclusivamente em
  `REQUEST_STATE_SECRET`. O TTL é de 15 minutos. O payload contém tudo que o
  servidor precisa para o retry, então o retry continua válido depois de um
  restart do MCP; adulteração e expiração retornam `-32602`.
- Reservas e Tasks são mantidas em memória, como permitido pelo enunciado.
  `dados/reservas.json` é a carga inicial e não é alterado; um restart volta a
  essa carga inicial. A integridade do `requestState` não depende dessa memória.
- O agente descobre as tools com `tools/list` antes do primeiro `tools/call`,
  lê `politica://uso` e extrai `2026-11-01` da primeira linha. O servidor MCP
  calcula política, conflitos e alternativas; o agente apenas traduz protocolo
  e estados.
- O parser A2A é determinístico e aceita somente a gramática definida:
  `reservar sala=... inicio=... fim=... responsavel=...` e
  `escolha=...`. Não há LLM, streaming, sessão de protocolo ou autenticação.

## Validação da entrega

Execute toda a bateria com um único comando na raiz do projeto:

```bash
python tests/run_all.py
```

O script usa somente a biblioteca padrão, gera um segredo temporário, inicia
os dois servidores em portas locais livres, roda os testes e encerra todos
os processos que iniciou. Não exige servidores previamente iniciados.

Execução em 03/10/2026: 16 testes locais, 36 verificações do validador e
4 verificações adicionais de integração passaram sem falhas. A saída completa
está em `tests/resultado-validacao.txt`.

Os testes locais verificam argumentos adulterados sem gravação adicional,
recusa, cancelamento, estado inválido, expirado ou assinado com outra chave,
uso do estado em outra ferramenta, alternativa ocupada após a pergunta,
repetição de retry, chamadas simultâneas, limites da política, intervalos
adjacentes, headers MCP, gramática A2A, erros e privacidade da Task.

As verificações adicionais confirmam `traceparent` no log MCP, retomada com
um token emitido antes de um reinício real do MCP usando o mesmo segredo,
resposta UTF-8 com tamanho HTTP correto e preservação dos arquivos de
`dados/`, `validador/` e `exemplos/`.

A revisão de código também confirmou que o agente se comunica com o MCP por
HTTP/JSON-RPC, descobre as tools e lê a política; ele não importa as funções
de domínio do servidor. O `requestState` é guardado como valor opaco e não é
copiado para a Task pública ou para o artifact.

O enunciado completo não acompanha este repositório. Esta validação cobre o
contrato do validador fornecido, as regras da política, as exigências descritas
neste README e o feedback do professor sobre o retry.

## Correções

O `inputResponses` de `reservar_sala`  é processado em um ramo exclusivo,
antes do fluxo normal de reserva. O servidor valida o `requestState` e usa
somente seus argumentos passados substituindo a sala pela alternativa aceita.
Um retry com `sala-mirante`, 13h às 14h e responsável `Biff` não grava esses
valores quando o estado foi emitido para outro pedido.

A consulta de conflito e a gravação são protegidas. Se a
alternativa já estiver ocupada na retomada, a chamada retorna `isError` sem
gerar outra reserva. Recusa e cancelamento concluem sem gravar; estados
inválidos retornam erro de protocolo `-32602`.

## Restrições não negociáveis

`dados/`, `validador/` e `exemplos/` continuam intactos. Não há banco de
dados, container, frontend, callback síncrono de elicitation, callback de
servidor para o cliente, OAuth ou dependência de modelo de linguagem.


## LOG de Saída do validador


PASS 01 tools/list traz as tres tools
PASS 02 toda tool tem inputSchema de objeto
PASS 03 listar_salas devolve structuredContent e o mesmo JSON em texto
PASS 04 _meta sem protocolVersion devolve -32602 e HTTP 400
PASS 05 _meta sem clientCapabilities devolve -32602 e HTTP 400
PASS 06 tool inexistente e recusada, por -32602 ou por isError
PASS 07 resources/read de politica://uso devolve a politica
PASS 08 resources/read de URI inexistente devolve -32602
PASS 09 sala inexistente devolve isError com a mensagem exata
PASS 10 fora da janela devolve isError com a mensagem exata
PASS 11 duracao acima de 2h devolve isError com a mensagem exata
PASS 12 intervalo invertido devolve isError com a mensagem exata
PASS 13 conflito devolve input_required com inputRequests e requestState
PASS 14 a elicitation e form mode e oferece as alternativas na ordem certa
PASS 15 conflito sem a capability elicitation devolve -32021 e HTTP 400
PASS 16 retry com inputResponses e requestState conclui a reserva
PASS 17 requestState adulterado e rejeitado com -32602
PASS 18 argumentos adulterados no retry nao tomam efeito
PASS 19 recusa conclui sem reservar e sem isError
PASS 20 conflito sem alternativa possivel devolve isError com a mensagem exata

PASS 21 agent card responde 200 no well-known com JSON
PASS 22 o card declara a interface JSON-RPC com url e versao 1.0
PASS 23 o card declara a skill reservar-sala
PASS 24 SendMessage com sala livre conclui a Task
PASS 25 o artifact chama reserva e traz a versao da politica
PASS 26 GetTask devolve id, contextId e estado corrente
PASS 27 SendMessage com sala ocupada pausa a Task
PASS 28 a Task pausada lista as alternativas na ordem certa
PASS 29 escolha fora do enum mantem a Task pausada
PASS 30 a continuacao conclui a Task na sala escolhida
PASS 31 SendMessage em Task terminal e recusado
PASS 32 a recusa termina a Task em CANCELED
PASS 33 duas Tasks pausadas ao mesmo tempo concluem cada uma com a sua reserva
PASS 34 nenhuma resposta A2A carrega o requestState
PASS 35 sala inexistente termina a Task em FAILED com a mensagem da tool
PASS 36 o agente e deterministico: o mesmo pedido produz a mesma pausa

resumo: 36 passaram, 0 falharam, de 36 verificacoes

