# doca

Um comando e você está dentro de um container dedicado ao projeto, com a versão certa da
linguagem — sem instalar nada no seu computador.

```bash
cd meu-projeto
doca
```

```
node@meu-projeto:/workspace$ node --version
v22.21.1
node@meu-projeto:/workspace$ pnpm install     # roda no container; o node_modules fica na sua pasta
```

Um único script POSIX `sh`. Precisa do Podman + `podman-compose` (preferido) ou, só na falta do
Podman, do Docker + Docker Compose. Funciona em qualquer terminal: bash do sistema, VSCode,
Cursor, e também dentro do **VSCode Flatpak**.

---

## Índice

- [O que o doca resolve](#o-que-o-doca-resolve)
- [Requisitos](#requisitos)
- [Instalação](#instalação)
- [Uso](#uso)
- [Criando o `doca.yml`](#criando-o-docayml)
- [Convenções obrigatórias](#convenções-obrigatórias)
- [Exemplos](#exemplos)
- [Como funciona](#como-funciona)
- [Segurança](#segurança)
- [Problemas comuns](#problemas-comuns)
- [O que mudou entre versões](#o-que-mudou-entre-versões)
- [Licença](#licença)

---

## O que o doca resolve

**Você entra no ambiente com um comando.** `doca` sobe os containers, espera os serviços ficarem
prontos e te entrega um shell dentro do projeto. Sair (`exit`) não desliga nada; `doca stop`
pausa e `doca down` apaga os containers (seus volumes ficam) quando você quiser.

**A versão da linguagem é a que você declara — e não muda sozinha.** O `doca.yml` fixa a imagem
(`node:22`, por exemplo), e uma trava impede que o gerenciador de pacotes seja baixado da
internet numa versão incompatível. Você recebe o pnpm/yarn que veio com aquele Node, como numa
instalação local.

**Nada suja o seu computador.** Cache, store do pnpm e histórico do shell vão para um volume
isolado. O código e o `node_modules` ficam na sua pasta (para o editor enxergar), e você apaga
com `rm -rf` quando quiser. `doca nuke` remove todo o ambiente sem tocar no seu código.

**Um pacote malicioso não alcança o resto da máquina.** O código executa dentro do container,
que só enxerga a pasta do projeto — não as suas chaves SSH, credenciais ou outros projetos.

**Cada projeto tem a sua versão e os seus recursos.** Um projeto em Node 22, outro em Node 24,
cada um com seu limite de CPU e memória, seus serviços (MySQL, RabbitMQ...) e seu banco isolado.

**Roda de qualquer terminal, inclusive VSCode Flatpak.** O script atravessa o sandbox sozinho —
você não configura nada para isso.

**Usa o Podman sempre que ele existe.** O Docker só entra se não houver Podman na máquina. A
detecção é automática, e o `doca.yml` é o mesmo nos dois casos.

A ideia é uma divisão de responsabilidades:

| Peça | Cuida de |
|---|---|
| **`doca`** (script global) | O que depende **da máquina**: sandbox, engine, SELinux, userns, bind do código, cache, fuso, ciclo de vida |
| **`doca.yml`** (um por projeto) | O que depende **do projeto**: imagem, versões, recursos, portas, serviços, volumes de dado |

O `doca.yml` não cita nada da máquina — por isso o **mesmo arquivo**, versionado no Git, sobe
igual no Zorin/Ubuntu, no Fedora e no Bazzite. Quem descobre em que sistema está e se adapta é
o script.

---

## Requisitos

- **Linux**
- **Podman** (rootless) + `podman-compose`. Com o Podman instalado, o `podman-compose` é
  obrigatório: o `doca` usa o Podman mesmo que o Docker também exista, e não recorre ao
  `docker compose`. Em distro imutável (Bazzite, Silverblue), instale o `podman-compose` pelo
  `brew` — nada vai para `/usr`
- **Docker** + Docker Compose, só se a máquina não tiver Podman
- **cgroups v2** com controladores delegados, para que os limites de CPU/memória valham

Não precisa de Node, Python ou qualquer runtime instalado no host.

### Onde foi testado

Todos os testes foram feitos com **Podman rootless** e com imagens prontas (`image:`):

| Sistema | podman-compose |
|---|---|
| Zorin OS 18.1 / Ubuntu 24.04 | 1.0.6 (apt) |
| Bazzite 44 / Fedora, SELinux em enforcing | 1.6 (Homebrew) |

Confira tudo de uma vez com `doca doctor`:

```
doca 1.1.0

sandbox    nenhum
engine     podman version 5.8.7
compose    podman-compose version 1.6.0
socket     unix:///run/user/1000/podman/podman.sock
userns     keep-id (só no app)
selinux    enforcing (bind do código com :z)
fuso       America/Sao_Paulo
cgroup     cpu io memory pids (limites valem)

projeto    /home/user/projetos/meu-projeto
arquivo    doca.yml
```

---

## Instalação

```bash
git clone https://github.com/DiegoSouza7/Doca.git ~/ferramentas/doca
cd ~/ferramentas/doca
./doca install
```

Abra um terminal novo e rode `doca doctor`.

### O que o `install` faz

Exatamente duas coisas, ambas dentro do seu `$HOME`:

1. Cria o atalho `~/.local/bin/doca` apontando para o script (criando `~/.local/bin` se não
   existir).
2. Acrescenta um bloco ao seu `~/.bashrc` (ou ao `~/.profile`, se não houver `~/.bashrc`):

```bash
# >>> doca >>>
export PATH="$HOME/.local/bin:$PATH"
# <<< doca <<<
```

Não pede `sudo`, não toca em `/usr`, não instala pacote, não copia o script (o atalho aponta
para o clone — você edita lá e o efeito é imediato).

> **Por que mexer no PATH?** Em Ubuntu e derivados, o `~/.local/bin` só entra no PATH se a pasta
> **já existir no momento do login**. Como o `install` acabou de criá-la, sem essa linha você
> precisaria reiniciar a sessão.

### Desinstalação

```bash
doca uninstall
```

Remove o atalho e apaga **apenas** o bloco entre as marcas `>>> doca >>>`. Containers e volumes
não são tocados — rode `doca nuke` em cada projeto antes, se quiser limpar tudo.

### Notas

- A **primeira** instalação precisa ser `./doca install` (ainda não há atalho no PATH). Depois,
  `doca install` funciona.
- Se você **mover a pasta**, o atalho quebra. Vá ao novo local e rode `./doca install` de novo.

---

## Uso

| Comando | O que faz |
|---|---|
| `doca` | Sobe o que falta e te põe dentro. Se já está de pé, entra na hora |
| `exit` | Sai do shell. **O ambiente continua ligado** |
| `doca exec S CMD` | Roda `CMD` no serviço `S` e sai |
| `doca logs [S]` | Acompanha os logs |
| `doca stop` | Pausa tudo — **containers preservados** |
| `doca down` | **Apaga os containers** — volumes preservados |
| `doca up` | Sobe sem entrar |
| `doca rebuild` | Recria os containers com a imagem que já está na máquina, sem entrar |
| `doca update` | Apaga os containers, baixa imagens novas e sobe — **pode trocar versões** |
| `doca nuke` | Apaga o ambiente — pergunta antes dos dados |
| `doca doctor` | Diagnóstico: engine, socket, limites, projeto |
| `doca help` | Ajuda completa |

### Fluxo típico

```bash
cd meu-projeto
doca                  # entrou
pnpm install
pnpm dev              # acesse na porta que o doca.yml publica
# Ctrl+C
exit                  # saiu; o ambiente continua ligado
doca stop             # pausou; amanhã o `doca` religa em segundos
```

### `stop` vs `down`

Os dois desligam o ambiente e nenhum dos dois toca em volume, código ou `node_modules`. A
diferença é o que acontece com o container em si — e ela é grande:

- **`stop`** só **para** os containers; eles continuam existindo. Religar (`doca`) leva segundos e
  você encontra tudo como deixou, inclusive o que instalou lá dentro fora dos volumes (um
  `apt install`, um arquivo em `/tmp`). É o desligar do dia a dia.
- **`down`** **apaga** os containers — não é só parar. A próxima subida cria containers novos a
  partir da imagem, e o que estava dentro do container fora dos volumes se perde. **Os volumes
  sobrevivem** (banco, cache, store do pnpm), assim como seu código. Use quando quiser um
  container limpo ou depois de mexer no `doca.yml` (ver
  [Editou o `doca.yml`?](#editou-o-docayml)).

Na dúvida, `stop`. O `down` é para quando você quer o container do zero.

### Editou o `doca.yml`?

```bash
doca down && doca
```

O `doca` com o ambiente de pé só entra no shell — não relê o `doca.yml`. O `down` apaga os
containers, e a próxima subida cria containers novos já com o arquivo editado.

### `rebuild` vs `update`

São separados de propósito:

- **`rebuild`** faz `down` e sobe de novo forçando a recriação, **sem baixar nada**: usa a
  imagem que já está na máquina. O efeito é o mesmo de `doca down && doca up`.
- **`update`** faz `down`, **baixa as imagens de novo** (`pull`) e sobe. Uma etiqueta como
  `node:22` é **alvo móvel**: hoje aponta para 22.21.1, semana que vem para 22.23.1 — e o pnpm
  que vem embutido muda junto. Por isso atualizar é um ato explícito, para você controlar quando
  as versões mudam.

Nenhum dos dois reconstrói imagem de `build:` — o `doca` não passa `--build` ao Compose. Se o
seu `doca.yml` usa `build:` com um `Dockerfile`, a imagem montada da primeira vez continua
sendo usada. O `doca` foi testado só com imagens prontas (`image:`), e é o recomendado.

### Ciclo de vida

```
                                      PAUSADO
                                         ⇅  ( doca stop / doca )
   NÃO EXISTE  ──────( doca )──────▶  LIGADO  ⇄  SEM CONTAINERS
        ▲                                   ( doca down / doca )
        │
        └──────────( doca nuke )────────────┘
```

**PAUSADO**: os containers existem, só estão parados. **SEM CONTAINERS**: o `down` apagou os
containers e **manteve os volumes** — a próxima subida cria containers novos com os mesmos dados.
**NÃO EXISTE**: nem containers, nem volumes (só o `nuke` chega aqui, e ele pergunta antes).
Ver [`stop` vs `down`](#stop-vs-down).

**Nunca afetado por nenhum comando:** seu código, `node_modules`, artefatos de build, arquivos
de ambiente e o `.git`. Nada disso é volume — está tudo no seu disco.

### Por que gerenciar só funciona de fora

Rodar `doca down` **dentro** do container mostra um aviso, não executa. É uma decisão de
segurança: para funcionar lá dentro, o container precisaria de acesso ao socket do engine — e
quem tem esse acesso pode criar um container privilegiado e escapar para o host.

**Lá dentro você trabalha; aqui fora você gerencia.**

---

## Criando o `doca.yml`

O `doca.yml` é um arquivo Compose comum. Qualquer coisa válida no Compose vale aqui.

O menor exemplo possível:

```yaml
name: meu-projeto

services:
  app:
    image: docker.io/library/node:22
```

Só isso. O `name:` vira o hostname do container e o prefixo dos volumes; o resto o `doca` põe.

### O que o `doca` injeta

Antes de chamar o Compose, o script gera um segundo arquivo Compose (o *overlay*) e passa os dois:
`-f doca.yml -f <overlay>`. O Compose mescla — é Compose puro, sem formato próprio. O overlay é
um arquivo temporário fora do projeto, apagado quando o `doca` termina; nada é gravado na pasta
do projeto. (Tem que ser arquivo: o podman-compose 1.0.6 não lê `/dev/fd` nem stdin.)

**No serviço `app`:**

| O quê | Valor | Por quê |
|---|---|---|
| Bind do código | `<pasta real>:/workspace` (`:z` com SELinux) | Código e `node_modules` ficam no seu disco, para o editor enxergar |
| `home:/home/node` | volume | Cache, store do pnpm e histórico fora do seu disco |
| `userns_mode` | `keep-id` no Podman, `host` no Docker | No Podman, o processo roda com o seu UID e os arquivos criados pelo container saem com o **seu** dono. No Docker, o `doca` não define `user:` — vale o usuário padrão da imagem |
| `command` | `sleep infinity` | O container **não roda a aplicação**: fica de pé esperando você. Quem roda `pnpm dev` é você |
| `working_dir`, `hostname` | `/workspace`, o `name:` | |
| `PATH` | começa com `/home/node/.local/bin` | Onde o corepack instala pnpm/yarn |
| `COREPACK_DEFAULT_TO_LATEST=0` | | A trava: vale o gerenciador que veio com este Node, não o mais recente publicado |
| `COREPACK_ENABLE_DOWNLOAD_PROMPT=0` | | Sem "continuar? [Y/n]" no meio do comando |
| `npm_config_store_dir` e `pnpm_config_store_dir` | `/home/node/.pnpm-store` | Store fora do projeto. O pnpm ≤ 10 lê a primeira; o 11 só a segunda |
| Caches (`XDG_*`, `npm_config_cache`, `YARN_CACHE_FOLDER`, `PNPM_HOME`) | sob `/home/node` | |
| `DOCA_INSIDE` | o `name:` | Só existe dentro — scripts podem checar onde estão |
| `TZ` | o fuso do host | Data do container igual à sua |

**Nos demais serviços:** só o `TZ` do host. Banco, cache e fila rodam como a imagem manda — os
entrypoints oficiais (MySQL, RabbitMQ, Redis) começam como root e acertam o dono do próprio
volume. Não declare `userns_mode` neles.

Se você declarar no `doca.yml` algo que o overlay também define, o overlay vence (variáveis de
ambiente são somadas).

### Imagem com nome completo

Use `docker.io/library/node:22`, não `node:22`. O Fedora/Bazzite vem com
`short-name-mode = "enforcing"`: sem um terminal para perguntar o registry, ele recusa nome
curto. Com o nome completo, o mesmo arquivo funciona em qualquer distro.

### Limites de recursos

```yaml
cpus: 4                    # no nível do serviço
mem_limit: 8g
```

Confirme que os limites valem de verdade:

```bash
doca exec -T app cat /sys/fs/cgroup/memory.max   # 8g → 8589934592
doca exec -T app cat /sys/fs/cgroup/cpu.max      # 4 cpus → 400000 100000
```

### Volumes

```yaml
volumes:
  db:        # dados — o nuke pergunta antes
```

O volume `home` (cache e store) quem cria é o `doca` — o `nuke` apaga sem perguntar. No
`doca.yml` vão só os volumes de dado.

**Não coloque `name:` fixo** nos volumes. Sem ele, o Compose prefixa com o nome do projeto
(`meu-projeto_home`, `meu-projeto_db`) — é o que torna o mesmo arquivo copiável entre projetos
sem colisão.

O `nuke` trata por convenção: qualquer volume terminado em `_home` é cache e some direto;
qualquer outro é dado e pergunta antes.

---

## Convenções obrigatórias

O `doca` espera quatro coisas do seu `doca.yml`. Elas não são configuráveis hoje.

### 1. Um `name:` no topo, sem aspas nem comentário

```yaml
name: meu-projeto
```

O `doca` lê a primeira linha que começa com `name:` e usa o resto dela **literalmente** — é com
esse valor que ele acha os containers e volumes do projeto. `name: "meu-projeto"` ou
`name: meu-projeto  # api` levam as aspas ou o comentário junto, e os comandos deixam de
encontrar o ambiente. Sem `name:`, o `doca` recusa rodar.

### 2. O serviço principal chama-se `app`

```yaml
services:
  app:            # ← este nome exato
```

É o serviço em que `doca` entra e onde `doca exec app ...` roda. Os outros serviços podem ter
qualquer nome.

### 3. O `HOME` do container é `/home/node`

O `doca` monta o volume `home` em `/home/node` e é ali que instala os atalhos do corepack,
escreve o `.bashrc` e guarda o cache. Imagens oficiais do Node já têm o usuário `node` (UID
1000) com esse HOME.

### 4. A imagem tem `bash`

`doca` abre o shell com `bash`. Imagens Alpine não o têm por padrão — use as variantes Debian
(`node:22`, não `node:22-alpine`), ou instale o `bash` na imagem.

> **Por que não é configurável?** Para manter o script simples e sem formato próprio. Se você
> precisa de outra stack (Python, Go) ou de Alpine, as partes que assumem Node falham em silêncio
> e o resto funciona — mas o shell e o cache não serão preparados. Abra uma issue se precisar.

---

## Exemplos

A pasta [`exemplos/`](exemplos/) tem arquivos completos e copiáveis:

| Arquivo | Cenário |
|---|---|
| [`nextjs.yml`](exemplos/nextjs.yml) | Front-end puro, sem serviços |
| [`api-mysql.yml`](exemplos/api-mysql.yml) | API + MySQL com healthcheck |
| [`api-multi-servicos.yml`](exemplos/api-multi-servicos.yml) | API + MySQL + Redis + RabbitMQ |

Copie para a raiz do projeto como `doca.yml` e ajuste o `name:`, a imagem e as portas.
Os três passam no `config` do podman-compose 1.0.6 e 1.6, já mesclados com o overlay do
`doca`.

### Como os serviços se enxergam

Dentro de um mesmo `doca.yml`, os serviços se acham **pelo nome**, na rede que o Compose cria
sozinho:

```yaml
environment:
  DB_HOST: mysql             # o nome do serviço
  DB_PORT: "3306"            # a porta interna, não a publicada
  AMQP_URL: amqp://guest:guest@rabbitmq:5672
```

Você **não precisa publicar portas** para isso funcionar. Só publique (`ports:`) o que você
quer alcançar do seu navegador ou de um cliente no host — a aplicação, o painel do RabbitMQ, o
banco para o seu editor de SQL.

### Sobrescrevendo o `.env`

Um caso comum: o `.env` aponta para `localhost` (porque funciona quando você roda no PC), mas
dentro do container `localhost` é o **próprio container**. Sobrescreva no `doca.yml`:

```yaml
env_file: [".env"]
environment:
  MYSQL_HOST: mysql        # vence o localhost do .env
  MYSQL_PORT: "3306"       # vence a porta publicada do .env
```

`environment` tem precedência sobre `env_file`.

### Projetos que conversam entre si

Dois projetos (uma API e um front, por exemplo) ficam em `doca.yml` separados, em redes
separadas — e por isso **não** se acham pelo nome. Duas saídas:

**1. Pela porta publicada no host** (zero mudança no código):

```yaml
extra_hosts: ["localhost:host-gateway"]
```

Isso faz o nome `localhost` apontar para o seu PC em vez de para o container. Se o front usa
`API_URL=http://localhost:3333` tanto no navegador quanto no servidor, é isto que faz funcionar.
Atenção: só vale para o **nome** `localhost` — um IP literal (`127.0.0.1`) não passa por aqui.

**2. Por uma rede compartilhada** (mais limpo, mas exige separar URL interna de externa no
código):

```yaml
networks:
  default:
    name: minha-rede-compartilhada
    external: true
```

---

## Como funciona

### Descoberta do projeto

O `doca` sobe as pastas até achar um `doca.yml`, igual o git procurando o `.git`. Por isso
`cd api && doca` cai no container certo sem você dizer qual.

### Detecção do engine

Procura **Podman primeiro**, depois Docker. A ordem importa: em muitas distros existe um binário
`docker` que é apenas um *wrapper* do Podman (pacote `podman-docker`). Procurar `docker` antes
encontraria o wrapper, concluiria "é Docker" e deixaria o `keep-id` de fora — e aí todo arquivo
criado pelo container sairia com dono errado, impossível de editar.

Achando o Podman, o compose é sempre o `podman-compose` — o `docker compose` não é usado,
mesmo que esteja instalado. O Docker (com `docker compose`) só entra quando não há Podman.

Esse é o **único** ponto do script que conhece a diferença entre os dois engines.

### Sandbox (Flatpak)

Se encontra `/.flatpak-info`, o script **se reexecuta no host** com `flatpak-spawn --host`, na
mesma pasta, e sai. Todo o resto é escrito como se Flatpak não existisse — e dentro do
container também não existe.

Isso importa mais do que parece: o `/tmp` que o VSCode Flatpak enxerga **não é** o `/tmp` do
sistema. São dois mundos com o mesmo nome. Como o script já roda no host quando entrega o
caminho ao engine, o caminho é o do host.

Antes de entregá-lo, o script ainda resolve o caminho real do projeto com `realpath`. É o que
transforma `/home/...` em `/var/home/...` no Fedora/Bazzite (ver a tabela abaixo).

### Debian × Fedora

O mesmo `doca.yml` sobe nos dois; o que muda, o script resolve:

| | Zorin / Ubuntu | Fedora / Bazzite |
|---|---|---|
| SELinux | não tem — o bind vai sem sufixo | em enforcing, o bind ganha `:z` |
| podman-compose | 1.0.6 (apt) | 1.6 (brew, no imutável) |
| `/home` | pasta comum | link para `/var/home` (o `realpath` resolve) |

**Por que `:z` e não `:Z`.** Sem rótulo, o SELinux nega ao container a leitura da pasta do
projeto (`EACCES` já no `package.json`). O `:z` rotula a pasta como `container_file_t`
*compartilhado*: o container lê e escreve, e o seu editor e o git — inclusive em Flatpak —
continuam lendo. O `:Z` seria privado de um container só. O script não usa `label=disable`
nem `chcon`, e só lê `/sys/fs/selinux/enforce` — nada é gravado no sistema.

### Por que o `keep-id` fica só no `app`

O `app` é o único que escreve na sua pasta, e o `keep-id` é o que faz esses arquivos saírem com
o seu dono. Nos outros serviços ele atrapalharia: com `keep-id` o Podman roda o processo como o
seu UID, e o MySQL não consegue mais ler o próprio volume (`binlog.index: Permission denied`).
Sem ele, os entrypoints oficiais começam como root e acertam o dono do volume sozinhos.

### Guard de produção

Em **todo comando de projeto** (`doca`, `up`, `exec`, `logs`, `stop`, `down`, `rebuild`,
`update`, `nuke`), o `doca` lê `.env` e `.env.local` procurando `NODE_ENV` ou
`NEXT_PUBLIC_APP_ENV`. Se algum declarar algo diferente de `development`, ele recusa:

```
✖ BLOQUEADO: o doca é exclusivo de desenvolvimento.
   .env declara 'production'.
```

Isso vale também para `stop`, `down` e `nuke`: com o `.env` em `production`, nenhum deles
roda. Volte o valor para `development` antes de gerenciar o ambiente. `doctor`, `help`,
`install` e `uninstall` não passam pelo guard.

Como a leitura funciona:

- Só conta a linha que **começa** com a variável (espaços antes são aceitos). Linhas comentadas
  (`# NODE_ENV=...`) e linhas com `export` na frente não são lidas.
- Do valor saem aspas e espaços, e o resto é comparado com `development` **literalmente**. Um
  comentário no fim da linha (`NODE_ENV=development # local`) vira `development#local` e
  bloqueia.

Se você precisa simular produção, `export NODE_ENV=production` **dentro** do container funciona
normalmente — o guard só lê os arquivos.

### Contornos do podman-compose

O `podman-compose` 1.0.6 implementa o Compose só parcialmente. O `doca` contorna estas coisas
para você não precisar saber que elas existem. Os contornos rodam com qualquer engine, não só
com o Podman:

| Recurso | Comportamento no podman-compose | Contorno |
|---|---|---|
| `up --wait` | Não existe | Espera manual: consulta o engine a cada segundo até nenhum container estar `starting` |
| `depends_on: service_healthy` | Vira `--requires`, que só ordena o start e **não espera a saúde** | Mesma espera manual |
| Código de saída | Devolve `0` mesmo quando o container falha ao iniciar | Confere o estado real via `ps` do engine |
| Religar depois do `stop` | Não é confiável no `up` | Roda `start` antes do `up -d` quando há containers parados |
| `-f /dev/fd/...` ou stdin | Não abre | O overlay é um arquivo temporário de verdade |

A espera manual falha na hora se algum container estiver `Exited`, `Dead` ou `unhealthy`, e
desiste depois de 90 segundos com `starting`.

Por isso vale declarar `healthcheck` nos serviços de dados: é ele que o `doca` usa para saber
que o banco está pronto antes de te entregar o shell. Sem `healthcheck`, o container nunca fica
`starting` e o `doca` não espera por ele.

---

## Segurança

**A proteção não vem de onde os arquivos estão, vem de onde eles são executados.** Um
`postinstall` é código arbitrário do autor do pacote — rodando no container, ele só tem o que o
container tem.

### O que o `doca` dá ao container

No serviço `app`, o overlay monta exatamente duas coisas:

- a pasta do projeto em `/workspace`;
- o volume `home` em `/home/node`.

Nada além disso é montado pelo `doca`: nem o seu `$HOME`, nem `~/.ssh`, nem outros projetos,
nem o socket do engine. Por isso o ciclo de vida não funciona lá dentro (ver
[Por que gerenciar só funciona de fora](#por-que-gerenciar-só-funciona-de-fora)). O que você
acrescentar em `volumes:` no `doca.yml` também fica visível para o container.

Sobre o usuário que roda o processo:

- **Podman**: rootless com `keep-id` — o processo roda com o seu UID e não pode mais que você.
- **Docker**: o `doca` injeta `userns_mode: host` e não define `user:`, então vale o usuário
  padrão da imagem.

Os limites de CPU e memória só existem se você declarar `cpus`/`mem_limit` no `doca.yml`.

### O que o container alcança

- **Tudo em `/workspace`**: pode alterar seu código e **ler seu `.env`** — senha de banco,
  credenciais de storage, tokens.
- **A rede**: o `doca` não restringe rede. O container sai para a internet e alcança o que a
  rede padrão do Compose alcança.

### O que roda no host a partir da pasta do projeto

O container pode escrever em qualquer arquivo de `/workspace`, e ela é a pasta do seu disco.
Todo programa **do host** que executa algo dessa pasta executa o que o container escreveu:

- **`.git/hooks`**: um `pre-commit` plantado ali roda no host no próximo commit. Para fechar,
  aponte `git config core.hooksPath` para fora do repositório.
- **Extensões do editor** que carregam código do projeto — ESLint (configuração e plugins de
  `node_modules`), Prettier, o TypeScript do workspace.
- **Arquivos de automação** que o host executa: `.vscode/tasks.json`, `.envrc` (direnv) e
  qualquer script que você rode no host.

Isso vale para qualquer ambiente em que você edite o código de fora do container. É o limite do
desenho, não um defeito dele.

### Por que `node_modules` fica no seu disco

`pnpm install` e `pnpm dev` rodam no container. Com o `node_modules` no disco, o editor enxerga
os tipos, o autocomplete funciona e você apaga com `rm -rf`. A contrapartida é a da seção
anterior: as ferramentas do host que carregam o `node_modules` executam esse código no host. O
que vai para volume é só o que você nunca quer ver: cache e store.

---

## Problemas comuns

**`doca: comando não encontrado` depois de mover a pasta**
O atalho guarda o caminho antigo. Vá ao novo local e rode `./doca install`.

**`A porta X já está em uso`**
Outro processo ou container está usando a porta — quando é um container, o `doca` diz qual.
Pare-o, ou mude a porta publicada no `doca.yml`.

**Um comando falha com "arquivo não encontrado", mas o arquivo existe**
Você provavelmente está **dentro** do container, onde o caminho do host não existe. `exit` e
rode no host. A variável `DOCA_INSIDE` só existe dentro do container — dá para checá-la em
scripts para impedir que rodem no lugar errado.

**Arquivos criados pelo container aparecem com dono errado**
No Podman, o `doca` injeta `keep-id` no `app`. Se o container não foi criado pelo `doca`,
recrie: `doca down && doca`. No Docker, o `doca` não define `user:` — o dono é o usuário padrão
da imagem.

**`Permission denied` no Fedora/Bazzite ao ler o projeto**
O container foi criado sem o `:z`. `doca down && doca` — o `doca doctor` deve mostrar
`selinux    enforcing (bind do código com :z)`.

**O RabbitMQ não sobe: `.erlang.cookie: eacces`**
O healthcheck rodou `rabbitmq-diagnostics` como root antes do servidor criar o cookie. Use
`["CMD", "gosu", "rabbitmq", "rabbitmq-diagnostics", "-q", "ping"]`. Se o volume já ficou com
o cookie do root, um `doca down && doca` resolve: o entrypoint acerta o dono ao subir.

**`short-name resolution enforced but cannot prompt without a TTY`**
A imagem está com nome curto. Use o nome completo: `docker.io/library/node:22`.

**O MySQL não sobe**
Se o seu `.env` tem `MYSQL_USER=root`, a imagem oficial recusa iniciar (`root` é reservado — a
variável serve para criar um usuário comum). Não use `env_file` no serviço do banco; passe só
`MYSQL_ROOT_PASSWORD` e `MYSQL_DATABASE` por `environment`.

**Aparece um `.pnpm-store` dentro do projeto**
O pnpm não viu a configuração do store e criou um na raiz do `/workspace`. O `doca` injeta
`npm_config_store_dir` (pnpm ≤ 10) e `pnpm_config_store_dir` (pnpm 11). Confira com
`doca exec -T app env | grep store_dir`; se as duas não aparecem, recrie o container com
`doca down && doca`. Depois apague a pasta `.pnpm-store` e reinstale.

**`corepack: EACCES permission denied`**
Ele tenta escrever em `/usr/local/bin`, que é do root. O `doca` já contorna instalando os
atalhos no HOME e pondo `/home/node/.local/bin` no começo do PATH. Um `PATH` declarado no
`doca.yml` é sobrescrito pelo overlay, então não interfere.

**Estou dentro ou fora do container?**
O prompt mostra `usuário@projeto:/workspace`. Ou cheque `$DOCA_INSIDE` — ela só existe dentro.

---

## O que mudou entre versões

### 1.1.0

- **O `doca` injeta o ambiente da máquina num overlay.** Bind do código, volume `home`,
  `userns_mode`, `PATH`, caches, `TZ` e o `:z` do SELinux saem do `doca.yml` e passam a vir do
  script (ver [O que o `doca` injeta](#o-que-o-doca-injeta)).
- **Novo comando `stop`**: pausa os containers sem apagá-los.
- **O formato da 1.0 deixou de ser aceito.** Na 1.0, o próprio `doca.yml` montava o código com
  `${DOCA_HOST_PATH}:/workspace`. Na 1.1 o `doca` não reconhece mais esse formato e aplica o
  overlay por cima dele — atualize o arquivo como abaixo.

#### Atualizando um `doca.yml` da 1.0

Apague do `doca.yml`:

- no `app`: `hostname`, `userns_mode`, `working_dir`, `command: sleep infinity`, `volumes`
  (código e `home`) e, do `environment`, `TZ`, `PATH`, `DOCA_INSIDE`, `COREPACK_*`,
  `npm_config_*`, `PNPM_HOME`, `XDG_*`, `YARN_CACHE_FOLDER`;
- nos demais serviços: `userns_mode` e `TZ`;
- em `volumes:` no fim: o `home`.

Se tiver RabbitMQ, troque o healthcheck para rodar como o usuário dele (ver
[`exemplos/api-multi-servicos.yml`](exemplos/api-multi-servicos.yml)).

Depois: `doca down && doca` — o `stop` não recria o container, e o bind novo só vale num
container novo. Os volumes de dado são preservados.

Atualize o `doca` **antes** de usar um `doca.yml` já no formato da 1.1: o script da 1.0 com o
YAML novo sobe o container sem o código montado.

---

## Licença

MIT.
