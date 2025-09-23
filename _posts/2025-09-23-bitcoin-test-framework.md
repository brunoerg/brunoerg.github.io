# O Framework de Testes Funcionais do Bitcoin Core

O Bitcoin Core, principal implementação do protocolo Bitcoin, é um software extremamente crítico: qualquer falha pode comprometer a segurança, a estabilidade ou o funcionamento da rede. Para reduzir riscos, o projeto conta com uma ampla infraestrutura de testes. Entre esses, destacam-se os testes funcionais, responsáveis por verificar o comportamento do sistema de ponta a ponta, simulando interações reais entre nós da rede.

## O que são testes funcionais?

Testes funcionais avaliam o sistema como um todo, em vez de partes isoladas do código. No contexto do Bitcoin Core, eles simulam situações reais — como envio e recebimento de transações, mineração de blocos, propagação de mensagens entre pares — e verificam se o software responde corretamente.
Esses testes complementam os testes unitários (voltados para funções e classes específicas), garantindo que as diferentes partes do Bitcoin Core funcionem de maneira integrada.

## Estrutura e linguagem

Os testes funcionais do Bitcoin Core são escritos em Python e ficam no diretório test/functional. Eles usam uma infraestrutura própria que facilita:

- Inicialização de múltiplos nós Bitcoin em um ambiente de teste isolado;

- Conexão entre esses nós simulando uma rede P2P real;

- Execução de comandos RPC (Remote Procedure Call) para controlar os nós (ex.: criar transações, gerar blocos);

- Validação de resultados verificando se o comportamento observado bate com o esperado.

## Como funcionam na prática?

Um teste funcional geralmente segue estas etapas:

1. Configuração do ambiente: inicializam-se um ou mais nós do Bitcoin Core, cada um com configurações específicas.

2. Execução de ações: os nós são instruídos a realizar operações, como enviar transações, conectar a peers ou minerar blocos.

3. Observação do comportamento: coleta-se o estado da rede ou o retorno das chamadas RPC.

4. Verificação: o teste compara o resultado obtido com o resultado esperado (assertivas). Se houver divergência, o teste falha.

Por exemplo, um teste pode verificar se, ao enviar uma transação inválida, o nó rejeita corretamente a mensagem em vez de propagá-la pela rede.

-----------

Para entender melhor o framework de testes functionais do Bitcoin Core podemos analisar o teste de exemplo que tem no repositorio, o `example_test.py` - que pode ser encontrado no diretorio `test/functional`.

## O que o `example_test` quer ensinar?

Esse arquivo é um modelo comentado que mostra como escrever um teste funcional que conversa com o nó tanto via RPC quanto via P2P: inicializa nós, conecta/desconecta pares, mina blocos, forja um bloco manualmente e o injeta pela interface P2P, valida alturas/sincronização e demonstra padrões de logging, sincronização e asserts. É exatamente o “hello world avançado” recomendado nas leituras introdutórias do framework.

## A anatomia do teste

### Importações e objetos P2P

O teste importa utilitários do `test_framework` (por ex., blocktools, messages, p2p, util) e define uma subclasse de P2PInterface chamada, p.ex., BaseNode. Essa subclasse sobrescreve callbacks como `on_block` para registrar blocos recebidos (num defaultdict) — assim o teste consegue observar o tráfego P2P e fazer assertivas sobre o que chegou e quantas vezes.

### Classe do teste

Todo teste herda de `BitcoinTestFramework` e normalmente sobrescreve:

- `set_test_params()`: define num_nodes, se a cadeia (blockchain) é limpa (`setup_clean_chain=True`) e flags extras de linha de comando por nó (p.ex., habilitar logs).

- `skip_test_if_missing_module()`: pula o teste se faltar um recurso (ex.: `self.skip_if_no_wallet()` quando o teste usa geração de blocos via wallet).

- `setup_network()`: quando você quer topologias não lineares, pode iniciar os nós, conectar apenas alguns pares e sincronizar só um subconjunto (ex.: conectar 0↔1, deixar o nó 2 isolado e chamar `self.sync_all(self.nodes[0:2])`).

#### Dica de RPC “mágico”

Acessos como `self.nodes[0].getblockcount()` funcionam porque o wrapper de nó envia chamadas RPC dinamicamente via __getattr__ — um detalhe que o `example_test` comenta explicitamente. Isso deixa o teste legível sem boilerplate de RPC.

Ou seja, você pode explicitamente chamar um RPC do nó 0, fazendo `self.nodes[0].chamadarpc(...parametros)`.

### O coração dos testes: `run_test()`

Dentro de `run_test()` o exemplo mostra vários padrões essenciais:

1. Abrir uma conexão P2P real

`peer = self.nodes[0].add_p2p_connection(BaseNode())`. O handshake espera verack, garantindo que a sessão está pronta antes de prosseguir. Agora você pode enviar mensagens P2P (ex.: send_message(msg_block)) e observar callbacks no seu BaseNode.

2. Sair do IBD e preparar estado

Minerar um bloco (“self.generate(self.nodes[0], nblocks=1, ...)”) tira o nó do Initial Block Download e sincroniza o par conectado. O example_test usa uma helper self.generate(...) para minerar e já sincronizar com um `sync_fun=`.

3. Criar e injetar um bloco pela rede P2P

O teste ilustra como construir um bloco manualmente com `create_block(...) + create_coinbase(...)`, resolver o proof-of-work (block.solve()), empacotar em `msg_block` e injetar direto no nó via `peer.send_message(msg_block)`. Isso permite exercitar caminhos que não passam pela mineração RPC, útil para casos negativos, limites de protocolo, etc.

4. Conectar um terceiro nó e sincronizar

Depois de fabricar/enviar blocos, o teste conecta node1 ↔ node2 e usa `self.sync_all()/waitforblockheight()` para esperar o estado consistente antes de afirmar alturas e contagens. Evitar flaky tests depende desses waits explícitos.

5. Concorrência nos objetos P2P

Como o network thread e o test thread compartilham estado, o exemplo protege leituras de buffers/contadores P2P com o p2p_lock ao checar que “cada bloco foi recebido exatamente uma vez”, finalizando com assert_equal(...). Esse padrão evita problemas de concorrência.

### Ciclo de vida do framework (um breve resumo)

1. **Setup**: parâmetros do teste, topologia, nós iniciados, conexões.

2. **Ações**: RPC (enviar txs, minerar, consultar estado) e/ou P2P (enviar mensagens cruas, espiar invs, headers, etc.).

3. **Sincronização/esperas**: sync_all, sync_blocks, sync_mempools, waitforblockheight, polling com wait_until.

4. **Asserts e logging**: assert_equal, assert_raises_rpc_error, self.log.info(...) para seccionar o teste e facilitar debug.

Essa filosofia — exercitar a aplicação end-to-end via RPC+P2P — é a base do framework.

----------

Em geral, o `example_test` é um guia prático de como pensar em testes funcionais no Core — montar cenários realistas (RPC) e validar efeitos observáveis (P2P), com sincronização explícita e cuidado com concorrência. Com este molde, dá para escrever desde testes simples até cenários de rede mais elaborados.

------------------

Se você esta lendo esse blog post porque vai participar do Bitcoin Students Day e, consequentemente, participar do Warnet, saiba que você nao precisa saber tudo isso. O principal é saber construir as mensagens (veja tudo relacionado a `msg_*`), as funções de comunicação - como `send_message` e, principalmente, ter noção dos comandos RPCs.
