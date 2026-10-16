# 1.3 — Redes para Engenharia DevOps

## Laboratório 1 — TCP/IP, IPv4, Sub-redes, Portas e Comunicação Cliente-Servidor

Este laboratório complementa a aula teórica **Fundamentos de Redes para Engenharia DevOps — Parte 1**. A proposta é observar como o Linux representa interfaces e endereços IP e, em seguida, acompanhar a comunicação entre processos utilizando **TCP** e **UDP**.

**Duração estimada:** 85 minutos de prática, com um desafio complementar opcional.

> **Ambiente:** Ubuntu no WSL2 ou outra distribuição Linux com Bash.**.

> **Isolamento:** todos os arquivos serão criados em `~/lab-devops-redes-01`. Execute os comandos no **mesmo terminal Bash**, pois algumas etapas utilizam as variáveis `LAB_DIR`, `TCP_PORT`, `UDP_PORT`, `TCP_PID` e `UDP_PID`. Se fechar o terminal, os processos iniciados a partir dele podem ser encerrados e essas variáveis serão perdidas.

---

## Pré-requisitos e validação do ambiente

As atividades utilizam ferramentas comuns do Linux e apenas a biblioteca-padrão do Python. Confira a disponibilidade:

```bash
command -v bash
command -v python3
command -v ip
command -v ss
command -v cat
command -v grep
command -v kill
```

Verifique o Python:

```bash
python3 --version
```

Os comandos `ip` e `ss` fazem parte do pacote **iproute2**. Em uma instalação Ubuntu na qual faltem ferramentas, instale os pacotes `iproute2` e `python3` **antes da aula**, seguindo os procedimentos da sua distribuição. Nenhuma etapa abaixo exige instalação de pacotes nem privilégios administrativos.

**Como interpretar a validação:** `command -v` deve exibir um caminho para cada ferramenta. Se algum comando não tiver saída, confirme sua instalação antes de continuar.

## Objetivos

Ao final da prática, você deverá ser capaz de:

- reconhecer interfaces de rede físicas, virtuais e de loopback;
- interpretar a representação de endereços IPv4 e prefixos CIDR;
- identificar blocos IPv4 e compreender a subdivisão de uma rede;
- diferenciar portas TCP e UDP e explicar o papel dos sockets;
- executar um servidor e um cliente na mesma máquina;
- observar sockets em escuta e conexões TCP estabelecidas;
- relacionar processos, endereços IP e portas ao funcionamento de aplicações.

---

# 1. Preparando o diretório do laboratório

Defina um diretório específico para esta aula:

```bash
export LAB_DIR="$HOME/lab-devops-redes-01"
mkdir -p -- "$LAB_DIR"
printf 'Laboratório: %s\n' "$LAB_DIR"
```

Confirme onde você está trabalhando:

```bash
ls -ld -- "$LAB_DIR"
```

**Atenção:** este material pressupõe um diretório recém-criado e que não foi utilizado anteriormente para outras atividades. Se já existir um laboratório incompleto nesse caminho, finalize os processos antigos antes de repetir os exercícios com os servidores.

---

# 2. Conhecendo as interfaces de rede do Linux

Interfaces de rede são pontos por meio dos quais o sistema operacional envia e recebe dados. Algumas correspondem a dispositivos de rede; outras são virtuais.

Liste as interfaces disponíveis:

```bash
ip -br link show
```

Observe as colunas de nome e estado. O nome das interfaces **varia de acordo com o ambiente**: é comum encontrar `lo` e outras interfaces com nomes como `eth0`, mas esses nomes não devem ser pressupostos.

Agora visualize os endereços IPv4 associados às interfaces:

```bash
ip -4 -br address show
```

Inspecione especificamente a interface de loopback:

```bash
ip -4 address show dev lo
```

Você deverá encontrar `127.0.0.1/8` entre seus endereços. A interface `lo` é utilizada para comunicação interna no próprio host. Esse endereço permite que dois processos troquem dados sem depender de uma conexão com outro computador.

**Questões**

1. Quantas interfaces estão disponíveis em seu ambiente?
2. Quais delas possuem endereços IPv4?
3. Por que `127.0.0.1` pode funcionar mesmo quando não há acesso à Internet?

> **WSL2:** os endereços observados são os da distribuição Linux, que pode utilizar rede virtualizada. Não espere encontrar necessariamente os mesmos endereços exibidos pelas configurações de rede do Windows.

---

# 3. Interpretando um endereço IPv4

Um endereço IPv4 possui **32 bits**, tradicionalmente apresentados em quatro octetos. Vamos observar a representação decimal e binária de um exemplo.

```bash
python3 - <<'PY'
from ipaddress import IPv4Address

ip = IPv4Address('192.168.10.25')
octetos = str(ip).split('.')

print('Endereço IPv4:', ip)
print('Octetos:      ', ' | '.join(octetos))
print('Representação binária:')
print(' '.join(format(int(octeto), '08b') for octeto in octetos))
PY
```

**Resultado esperado:**

```text
Endereço IPv4: 192.168.10.25
Octetos:       192 | 168 | 10 | 25
Representação binária:
11000000 10101000 00001010 00011001
```

Perceba que cada grupo binário tem 8 bits. O endereço completo tem 4 × 8 = 32 bits.

**Questão:** por que o valor de cada octeto varia de `0` até `255`?

---

# 4. Explorando sub-redes e prefixos CIDR

A notação CIDR identifica quantos bits de um endereço pertencem ao prefixo de rede. Neste exemplo, o prefixo `/24` reserva 24 bits para identificar o bloco.

```bash
python3 - <<'PY'
from ipaddress import IPv4Network

rede = IPv4Network('192.168.10.0/24')

print('Rede:              ', rede)
print('Máscara:           ', rede.netmask)
print('Endereço de rede:  ', rede.network_address)
print('Último endereço:   ', rede.broadcast_address)
print('Total de endereços:', rede.num_addresses)
print('Hosts utilizáveis: ', sum(1 for _ in rede.hosts()))
PY
```

**Resultado esperado:**

```text
Rede:               192.168.10.0/24
Máscara:            255.255.255.0
Endereço de rede:   192.168.10.0
Último endereço:    192.168.10.255
Total de endereços: 256
Hosts utilizáveis:  254
```

Em uma sub-rede IPv4 tradicional `/24`, normalmente um endereço representa a própria rede e outro é reservado para broadcast. Por isso, temos **256 endereços no bloco e 254 endereços utilizáveis por hosts**. Existem exceções em outros tipos de prefixo e ambiente, que não são o foco desta prática.

Vamos dividir o mesmo bloco em quatro sub-redes `/26`:

```bash
python3 - <<'PY'
from ipaddress import IPv4Network

rede = IPv4Network('192.168.10.0/24')

for subrede in rede.subnets(new_prefix=26):
    print(f'{str(subrede):20} | máscara: {str(subrede.netmask):15} | '
          f'endereços: {subrede.num_addresses:2} | '
          f'hosts: {sum(1 for _ in subrede.hosts()):2}')
PY
```

**Resultado esperado:**

```text
192.168.10.0/26      | máscara: 255.255.255.192 | endereços: 64 | hosts: 62
192.168.10.64/26     | máscara: 255.255.255.192 | endereços: 64 | hosts: 62
192.168.10.128/26    | máscara: 255.255.255.192 | endereços: 64 | hosts: 62
192.168.10.192/26    | máscara: 255.255.255.192 | endereços: 64 | hosts: 62
```

Por último, verifique a qual bloco um endereço pertence:

```bash
python3 - <<'PY'
from ipaddress import IPv4Address, IPv4Network

ip = IPv4Address('192.168.10.70')

for subrede in IPv4Network('192.168.10.0/24').subnets(new_prefix=26):
    if ip in subrede:
        print(f'{ip} pertence a {subrede}')
PY
```

**Resultado esperado:**

```text
192.168.10.70 pertence a 192.168.10.64/26
```

**Questões**

1. Qual a diferença entre a quantidade total de endereços de um bloco e a quantidade de endereços utilizáveis por hosts?
2. Por que um `/26` representa um bloco menor do que um `/24`?
3. Em qual das quatro sub-redes está o endereço `192.168.10.130`?

> A biblioteca `ipaddress` faz os cálculos por nós, mas não altera a configuração de rede do computador. Nenhuma sub-rede real será criada nesta etapa.

---

# 5. Criando um servidor de teste para TCP e UDP

Para relacionar protocolos, portas e aplicações, utilizaremos um programa muito pequeno em Python. Ele funcionará como um **servidor de eco (*echo server*)**: ao receber uma mensagem, enviará uma resposta contendo a mesma mensagem com um prefixo.

O mesmo programa poderá operar em modo TCP ou UDP. Os detalhes da API `socket` **não precisam ser memorizados**; o objetivo é observar o comportamento da comunicação.

Crie o arquivo:

```bash
cat > "$LAB_DIR/servidor.py" <<'PY'
#!/usr/bin/env python3
import socket
import sys
from pathlib import Path

if len(sys.argv) != 2 or sys.argv[1] not in ('tcp', 'udp'):
    raise SystemExit('Uso: python3 servidor.py tcp|udp')

protocolo = sys.argv[1]
base = Path(__file__).resolve().parent
arquivo_porta = base / f'{protocolo}.port'
tipo = socket.SOCK_STREAM if protocolo == 'tcp' else socket.SOCK_DGRAM

with socket.socket(socket.AF_INET, tipo) as servidor:
    servidor.bind(('127.0.0.1', 0))

    if protocolo == 'tcp':
        servidor.listen(5)

    porta = servidor.getsockname()[1]
    arquivo_porta.write_text(f'{porta}\n', encoding='ascii')
    print(f'[OK] Servidor {protocolo.upper()} em 127.0.0.1:{porta}', flush=True)

    if protocolo == 'tcp':
        while True:
            conexao, endereco = servidor.accept()
            with conexao:
                conexao.settimeout(30)
                try:
                    dados = conexao.recv(4096)
                except socket.timeout:
                    continue
                if dados:
                    conexao.sendall(b'TCP: ' + dados)
    else:
        while True:
            dados, endereco = servidor.recvfrom(4096)
            servidor.sendto(b'UDP: ' + dados, endereco)
PY
```

O que observar no código:

- `127.0.0.1` restringe a comunicação ao próprio host;
- `socket.SOCK_STREAM` utiliza TCP, enquanto `socket.SOCK_DGRAM` utiliza UDP;
- a porta `0` solicita ao sistema operacional uma porta local disponível, evitando a escolha de uma porta fixa que poderia já estar ocupada;
- TCP recebe conexões com `accept()`, enquanto UDP recebe mensagens com `recvfrom()`;
- o programa registra a porta escolhida em `tcp.port` ou `udp.port`, dentro do laboratório.

Verifique a sintaxe antes de executar:

```bash
python3 -m py_compile "$LAB_DIR/servidor.py"
```

Se estiver correta, a verificação não deverá emitir mensagens. O Python poderá criar um diretório `__pycache__` dentro do laboratório.

---

# 6. Iniciando e inspecionando o servidor TCP

Inicie o servidor em segundo plano. O PID será armazenado em `TCP_PID` para permitir o encerramento controlado posteriormente.

```bash
rm -f -- "$LAB_DIR/tcp.port"
python3 -u "$LAB_DIR/servidor.py" tcp > "$LAB_DIR/tcp.log" 2>&1 &
TCP_PID=$!
```

Aguarde a criação do arquivo com a porta e consulte o número atribuído:

```bash
for tentativa in {1..30}; do
    [[ -s "$LAB_DIR/tcp.port" ]] && break
    sleep 0.1
done

if [[ -s "$LAB_DIR/tcp.port" ]]; then
    TCP_PORT=$(cat "$LAB_DIR/tcp.port")
    printf 'Servidor TCP: PID=%s, porta=%s\n' "$TCP_PID" "$TCP_PORT"
else
    printf '[ERRO] Servidor TCP não iniciou. Consulte o log:\n' >&2
    cat "$LAB_DIR/tcp.log"
fi
```

**Se ocorrer um erro, interrompa a prática e confira o arquivo `tcp.log` antes de continuar.** O número da porta varia a cada execução.

Observe os sockets TCP que estão aguardando conexões:

```bash
ss -lnt "sport = :$TCP_PORT"
```

A saída deverá incluir um socket em estado **LISTEN** associado ao endereço `127.0.0.1` e à porta informada. O comando filtra pelo número da porta escolhida pelo sistema.

Agora envie uma mensagem utilizando um processo cliente:

```bash
python3 - "$TCP_PORT" <<'PY'
import socket
import sys

porta = int(sys.argv[1])
with socket.create_connection(('127.0.0.1', porta), timeout=5) as cliente:
    cliente.sendall(b'ola, TCP!')
    resposta = cliente.recv(4096)

print('Resposta:', resposta.decode('utf-8'))
PY
```

**Resultado esperado:**

```text
Resposta: TCP: ola, TCP!
```

Nesse exemplo, o **cliente** inicia uma conexão com o socket em escuta do servidor. Após receber a mensagem, o servidor envia uma resposta pela mesma conexão.

**Questões**

1. Qual processo está aguardando conexões e qual inicia a comunicação?
2. Qual é o papel de `127.0.0.1` e qual é o papel de `TCP_PORT`?
3. O número da porta do servidor precisa ser fixo para que exista comunicação TCP?

---

# 7. Observando uma conexão TCP estabelecida

O cliente anterior abre, utiliza e encerra a conexão rapidamente. Para visualizar o estado **ESTABLISHED**, criaremos uma conexão que ficará aberta por alguns segundos.

```bash
python3 -u - "$TCP_PORT" <<'PY' &
import socket
import sys
import time

porta = int(sys.argv[1])
with socket.create_connection(('127.0.0.1', porta), timeout=5):
    print('[INFO] Conexão TCP aberta por 15 segundos.', flush=True)
    time.sleep(15)
PY
CLIENT_PID=$!
```

Aguarde até que a conexão realmente seja estabelecida (a inicialização do processo pode levar alguns instantes) e então inspecione-a:

```bash
for tentativa in {1..30}; do
    if ss -tn "( sport = :$TCP_PORT or dport = :$TCP_PORT )" | grep -q 'ESTAB'; then
        break
    fi
    sleep 0.1
done

ss -tn "( sport = :$TCP_PORT or dport = :$TCP_PORT )"
```

A saída deverá mostrar, enquanto o cliente estiver conectado, a comunicação entre duas portas locais: a porta escolhida para o servidor e uma **porta de origem** atribuída ao cliente. Como ambos os processos estão no mesmo host, as duas extremidades utilizam `127.0.0.1`.

Observe que a combinação de IP e porta de origem e destino permite distinguir uma conexão TCP de outra. O estado **ESTABLISHED** representa uma conexão em andamento; **LISTEN** identifica um socket preparado para aceitar novas conexões.

Finalize a conexão de demonstração antes de continuar (se os 15 segundos já tiverem passado, não haverá processo ativo):

```bash
if kill -0 "$CLIENT_PID" 2>/dev/null; then
    kill "$CLIENT_PID"
fi
wait "$CLIENT_PID" 2>/dev/null || true
```

**Questão:** por que o cliente pode utilizar uma porta diferente a cada conexão, enquanto o servidor continua aguardando solicitações na mesma porta?

---

# 8. Iniciando e inspecionando o servidor UDP

O UDP não realiza o estabelecimento prévio de uma conexão equivalente à conexão TCP. Vamos reutilizar nosso servidor em modo UDP.

```bash
rm -f -- "$LAB_DIR/udp.port"
python3 -u "$LAB_DIR/servidor.py" udp > "$LAB_DIR/udp.log" 2>&1 &
UDP_PID=$!
```

Aguarde a porta ser registrada:

```bash
for tentativa in {1..30}; do
    [[ -s "$LAB_DIR/udp.port" ]] && break
    sleep 0.1
done

if [[ -s "$LAB_DIR/udp.port" ]]; then
    UDP_PORT=$(cat "$LAB_DIR/udp.port")
    printf 'Servidor UDP: PID=%s, porta=%s\n' "$UDP_PID" "$UDP_PORT"
else
    printf '[ERRO] Servidor UDP não iniciou. Consulte o log:\n' >&2
    cat "$LAB_DIR/udp.log"
fi
```

**Se ocorrer um erro, consulte `udp.log` antes de continuar.**

Inspecione o socket UDP disponível:

```bash
ss -lnu "sport = :$UDP_PORT"
```

Ao contrário do TCP, você não verá um estado **LISTEN** de conexão TCP. O UDP trabalha com datagramas e não estabelece o mesmo tipo de conexão orientada a fluxo.

Envie um datagrama UDP ao servidor:

```bash
python3 - "$UDP_PORT" <<'PY'
import socket
import sys

porta = int(sys.argv[1])
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as cliente:
    cliente.settimeout(5)
    cliente.sendto(b'ola, UDP!', ('127.0.0.1', porta))
    resposta, origem = cliente.recvfrom(4096)

print('Resposta:', resposta.decode('utf-8'))
print('Origem:', origem)
PY
```

**Resultado esperado:**

```text
Resposta: UDP: ola, UDP!
Origem: ('127.0.0.1', <porta UDP do servidor>)
```

A porta numérica exibida no campo `Origem` será aquela atribuída ao servidor UDP. O cliente enviou um datagrama diretamente, sem realizar o three-way handshake do TCP.

**Questões**

1. O que muda entre `ss -lnt` e `ss -lnu`?
2. Por que os dois exemplos conseguem transportar mensagens mesmo oferecendo serviços diferentes?
3. Uma resposta recebida nesse teste significa que o UDP garante a entrega de todos os datagramas? Justifique.

---

# 9. Comparando TCP e UDP no mesmo host

Agora você possui dois processos servidores utilizando o loopback. Observe-os juntos:

```bash
printf '%s\n' 'Sockets TCP:'
ss -lnt "sport = :$TCP_PORT"

printf '\n%s\n' 'Sockets UDP:'
ss -lnu "sport = :$UDP_PORT"
```

Veja também os PIDs dos processos que foram iniciados neste terminal:

```bash
printf 'Servidor TCP: PID %s\n' "$TCP_PID"
printf 'Servidor UDP: PID %s\n' "$UDP_PID"
ps -p "$TCP_PID" -p "$UDP_PID" -o pid=,args=
```

**Interpretação:** cada servidor criou um socket de um protocolo de transporte específico. Um número de porta TCP não é o mesmo recurso que uma porta UDP com o mesmo número; os protocolos mantêm seus próprios espaços de portas.

**Desafio complementar (opcional):** modifique apenas a mensagem enviada em um dos clientes e confirme que a resposta continua chegando pelo protocolo correspondente. Depois, explique qual parte do código do servidor identifica a diferença entre TCP e UDP.

---

# 10. Encerrando os processos de teste

Encerre somente os processos que foram iniciados neste laboratório e cujos PIDs estão armazenados nas variáveis desta sessão:

```bash
for pid in "$TCP_PID" "$UDP_PID"; do
    if kill -0 "$pid" 2>/dev/null; then
        kill "$pid"
    fi
done

wait "$TCP_PID" 2>/dev/null || true
wait "$UDP_PID" 2>/dev/null || true
```

Confira se os sockets dos servidores deixaram de aparecer:

```bash
ss -lnt "sport = :$TCP_PORT"
ss -lnu "sport = :$UDP_PORT"
```

A consulta deverá apresentar apenas os cabeçalhos, sem sockets correspondentes às portas selecionadas — desde que nenhum outro processo tenha reutilizado essas portas.

Os arquivos do laboratório poderão ser mantidos para revisão e para as próximas aulas. **Não é necessário apagá-los.**

### Limpeza opcional dos arquivos

Antes de excluir qualquer coisa, verifique o caminho:

```bash
printf '%s\n' "$LAB_DIR"
ls -la -- "$LAB_DIR"
```

Se tiver terminado e desejar remover **somente este diretório de laboratório**, utilize a verificação abaixo:

```bash
if [[ "$LAB_DIR" == "$HOME/lab-devops-redes-01" && -d "$LAB_DIR" && ! -L "$LAB_DIR" ]]; then
    rm -r -- "$LAB_DIR"
    printf '[OK] Diretório do laboratório removido.\n'
else
    printf '[ERRO] Caminho inesperado; nada foi removido.\n' >&2
fi
```

> **Importante:** execute essa limpeza apenas depois de confirmar que os dois servidores foram encerrados. Não utilize o comando de remoção se quiser preservar o programa para estudar ou reutilizar nas próximas aulas.

---

# 11. Checklist final

Ao concluir o laboratório, verifique se você consegue explicar:

- [ ] Por que um host pode ter mais de uma interface de rede e mais de um endereço IP.
- [ ] Qual a função do endereço `127.0.0.1` e da interface `lo`.
- [ ] Por que o IPv4 utiliza 32 bits e o que representa um prefixo CIDR.
- [ ] Como um bloco `/24` pode ser subdividido em quatro blocos `/26`.
- [ ] A diferença entre um endereço IP e uma porta de transporte.
- [ ] Por que um servidor precisa disponibilizar um socket para receber dados.
- [ ] Como um cliente TCP inicia uma conexão com um servidor.
- [ ] Por que TCP e UDP possuem comportamentos diferentes.
- [ ] O que significa encontrar um socket TCP em `LISTEN` ou uma conexão em `ESTABLISHED`.
- [ ] Como localizar sockets utilizando `ss` sem alterar a configuração de rede.

## Resultado esperado

Ao final desta prática, você terá observado a comunicação entre **dois processos na mesma máquina**, utilizando a pilha TCP/IP do Linux. A principal aprendizagem é relacionar **interface → endereço IP → protocolo de transporte → porta → socket → processo**.

Os próximos laboratórios abordarão outros aspectos das redes. Este exercício **não configura roteamento, NAT, firewalls, DNS, HTTPS, proxies ou balanceadores**, nem publica serviços na Internet.

---

# Referências

- **KUROSE, J. F.; ROSS, K. W.** *Computer Networking: A Top-Down Approach*. 8. ed. Pearson, 2021. Caps. 1–4 (fundamentos de redes, aplicações, transporte e endereçamento IPv4).
- **Documentação do Python 3:** biblioteca padrão `socket` e módulo `ipaddress`.
- **Documentação do Linux:** ferramentas `ip` e `ss` do pacote `iproute2`.