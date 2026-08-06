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
node@meu-projeto:/workspace$ pnpm install     # escreve no container, não no seu PC
```

Um script `sh` de ~550 linhas, sem dependências além do Podman ou Docker. Funciona em qualquer
terminal: bash do sistema, VSCode, Cursor, e também dentro do **VSCode Flatpak**.

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

**Funciona com Podman ou Docker sem você escolher.** Ele detecta qual está na máquina e se
adapta. Trocar de engine não pede mudança nenhuma no projeto.

A ideia é uma divisão de responsabilidades:

| Peça | Cuida de |
|---|---|
| **`doca`** (script global) | O que depende **da máquina**: sandbox, engine, caminhos, ciclo de vida |
| **`doca.yml`** (um por projeto) | O que depende **do projeto**: imagem, versões, recursos, serviços |

O script não sabe o que é Node, pnpm ou MySQL. Ele sobe qualquer `doca.yml`.

---

## Requisitos

- **Linux** (testado em Zorin OS 18 / Ubuntu 24.04)
- **Podman** (rootless, recomendado) + `podman-compose`, **ou** **Docker** + Docker Compose
- **cgroups v2** com controladores delegados, para que os limites de CPU/memória valham

Não precisa de Node, Python ou qualquer runtime instalado no host.

Confira tudo de uma vez com `doca doctor`:

```
doca 1.0.0

sandbox    nenhum
engine     podman version 4.9.3
compose    podman-compose version: 1.0.6
socket     unix:///run/user/1000/podman/podman.sock
userns     keep-id
cgroup     cpu memory pids (limites valem)

projeto    /home/user/projetos/meu-projeto
arquivo    doca.yml
```

---

## Instalação

```bash
git clone https://github.com/SEU_USUARIO/doca.git ~/ferramentas/doca
cd ~/ferramentas/doca
./doca install
```

Abra um terminal novo e rode `doca doctor`.

### O que o `install` faz

Exatamente duas coisas, ambas dentro do seu `$HOME`:

1. Cria o atalho `~/.local/bin/doca` apontando para o script (criando `~/.local/bin` se não
   existir).
2. Acrescenta um bloco ao seu `~/.bashrc`:

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
| `doca rebuild` | Recria os containers com a imagem que já está na máquina |
| `doca update` | Baixa imagens novas — **pode trocar versões** |
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
  container limpo ou depois de mexer no `doca.yml`.

Na dúvida, `stop`. O `down` é para quando você quer o container do zero.

### `rebuild` vs `update`

São separados de propósito:

- **`rebuild`** recria os containers com a imagem que já está na máquina. Use depois de editar
  o `doca.yml`.
- **`update`** baixa imagens novas. Uma etiqueta como `node:22` é **alvo móvel**: hoje aponta
  para 22.21.1, semana que vem para 22.23.1 — e o pnpm que vem embutido muda junto. Por isso
  atualizar é um ato explícito, para você controlar quando as versões mudam.

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
    hostname: meu-projeto
    userns_mode: ${DOCA_USERNS}
    working_dir: /workspace
    command: sleep infinity
    environment:
      PATH: /home/node/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
      DOCA_INSIDE: meu-projeto
    volumes:
      - ${DOCA_HOST_PATH}:/workspace
      - home:/home/node

volumes:
  home:
```

O `command: sleep infinity` é o coração da ideia: o container **não roda a aplicação**. Ele fica
de pé, vazio, esperando você digitar — como uma máquina dedicada ao projeto. Quem roda `pnpm dev`
é você.

### As duas variáveis que o `doca` injeta

Você **não** define estas — o script as exporta antes de chamar o Compose:

| Variável | O que é |
|---|---|
| `${DOCA_HOST_PATH}` | O caminho real do projeto no host. Necessário porque o que um sandbox enxerga nem sempre é o caminho real |
| `${DOCA_USERNS}` | `keep-id` no Podman, `host` no Docker. É o que faz os arquivos criados pelo container saírem com o **seu** dono |

### A trava de versão do gerenciador

```yaml
environment:
  # Sem esta linha, o corepack ignora o gerenciador que veio com este Node e
  # baixa o mais recente publicado — que pode não suportar a versão fixada
  # na imagem. Com ela, vale o que veio junto, como numa instalação local.
  COREPACK_DEFAULT_TO_LATEST: "0"

  # Sem isto, ele pergunta "continuar? [Y/n]" no meio do seu comando.
  COREPACK_ENABLE_DOWNLOAD_PROMPT: "0"
```

### Cache fora do seu disco

```yaml
environment:
  # store_dir é o mais importante: sem ele o pnpm cria um .pnpm-store DENTRO
  # do projeto (ao lado do node_modules, para usar hardlinks). Apontando para
  # o HOME (que é volume), o cache não toca o seu disco.
  npm_config_store_dir: /home/node/.pnpm-store
  PNPM_HOME: /home/node/.pnpm
  XDG_CACHE_HOME: /home/node/.cache
  XDG_DATA_HOME: /home/node/.local/share
  npm_config_cache: /home/node/.npm
  YARN_CACHE_FOLDER: /home/node/.yarn
```

### Fuso e limites de recursos

```yaml
environment:
  TZ: America/Sao_Paulo    # data igual à do host

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
  home:      # cache e store — o nuke apaga sem perguntar
  db:        # dados — o nuke pergunta antes
```

**Não coloque `name:` fixo** nos volumes. Sem ele, o Compose prefixa com o nome do projeto
(`meu-projeto_home`, `meu-projeto_db`) — é o que torna o mesmo arquivo copiável entre projetos
sem colisão.

O `nuke` trata por convenção: qualquer volume terminado em `_home` é cache e some direto;
qualquer outro é dado e pergunta antes.

---

## Convenções obrigatórias

O `doca` espera três coisas do seu `doca.yml`. Elas não são configuráveis hoje.

### 1. O serviço principal chama-se `app`

```yaml
services:
  app:            # ← este nome exato
```

É o serviço em que `doca` entra e onde `doca exec app ...` roda. Os outros serviços podem ter
qualquer nome.

### 2. O `HOME` do container é `/home/node`

```yaml
volumes:
  - home:/home/node     # ← este caminho exato
```

É onde o `doca` instala os atalhos do corepack, escreve o `.bashrc` e guarda o cache. Imagens
oficiais do Node já têm o usuário `node` com esse HOME.

### 3. A imagem tem `bash`

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
Os três foram validados subindo de verdade neste ambiente.

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

Esse é o **único** ponto do script que conhece a diferença entre os dois engines.

### Sandbox (Flatpak)

Se detecta que está dentro de um sandbox, o script **se reexecuta no host** e sai. É uma linha,
no topo. Todo o resto é escrito como se Flatpak não existisse — e dentro do container também
não existe.

Isso importa mais do que parece: o `/tmp` que o VSCode Flatpak enxerga **não é** o `/tmp` do
sistema. São dois mundos com o mesmo nome. Por isso o script resolve o caminho real do projeto
(`realpath`) antes de entregá-lo ao engine.

### Guard de produção

Antes de subir, o `doca` lê `.env` e `.env.local` procurando `NODE_ENV` ou
`NEXT_PUBLIC_APP_ENV`. Se algum declarar algo diferente de `development`, ele recusa:

```
✖ BLOQUEADO: o doca é exclusivo de desenvolvimento.
   .env declara 'production'.
```

Linhas comentadas são ignoradas. O guard só age ao subir — se você precisa simular produção,
`export NODE_ENV=production` **dentro** do container funciona normalmente.

### Contornos do podman-compose

O `podman-compose` 1.0.6 implementa o Compose só parcialmente. O `doca` contorna três coisas
para você não precisar saber que elas existem:

| Recurso | Comportamento no podman-compose | Contorno |
|---|---|---|
| `up --wait` | Não existe | Espera manual até nenhum container estar `starting` |
| `depends_on: service_healthy` | Vira `--requires`, que só ordena o start e **não espera a saúde** | Mesma espera manual |
| Código de saída | Devolve `0` mesmo quando o container falha ao iniciar | Confere o estado real via `podman ps` |

Por isso vale declarar `healthcheck` nos serviços de dados: é ele que o `doca` usa para saber
que o banco está pronto antes de te entregar o shell.

---

## Segurança

**A proteção não vem de onde os arquivos estão, vem de onde eles são executados.** Um
`postinstall` é código arbitrário do autor do pacote.

### O que ele NÃO alcança

- Suas chaves SSH, credenciais de nuvem, cookies, outros projetos, o resto do disco. No PC ele
  alcançaria tudo isso; no container, só a pasta do projeto.
- Mais que o seu usuário. Com Podman rootless + `keep-id`, o container nunca pode mais que você.
  Compare: Docker tradicional roda como **root no host**.
- Mais recursos que o permitido — desde que você use `cpus`/`mem_limit`.

### O que ele alcança

- **Tudo em `/workspace`**: pode alterar seu código e **ler seu `.env`** — senha de banco,
  credenciais de storage, tokens.
- **`.git/hooks`**: é a única fuga real. Um `pre-commit` plantado ali é executado pelo **seu
  host** no próximo commit. Para fechar, aponte `git config core.hooksPath` para fora do
  repositório.

Isso vale para qualquer ambiente em que você edite o código de fora do container. É o limite do
desenho, não um defeito dele.

### Por que `node_modules` fica no seu disco

Porque não muda nada em segurança — a execução acontece dentro do container de qualquer jeito —
e em troca o editor enxerga os tipos, o autocomplete funciona e você apaga com `rm -rf`. O que
vai para volume é só o que você nunca quer ver: cache e store.

---

## Problemas comuns

**`doca: comando não encontrado` depois de mover a pasta**
O atalho guarda o caminho antigo. Vá ao novo local e rode `./doca install`.

**`A porta X já está em uso`**
Outro ambiente está de pé com a mesma porta. Pare-o, ou mude a porta publicada.

**Um comando falha com "arquivo não encontrado", mas o arquivo existe**
Você provavelmente está **dentro** do container, onde o caminho do host não existe. `exit` e
rode no host. A variável `DOCA_INSIDE` só existe dentro do container — dá para checá-la em
scripts para impedir que rodem no lugar errado.

**Arquivos criados pelo container aparecem com dono errado**
Falta o `userns_mode: ${DOCA_USERNS}` no serviço.

**O MySQL não sobe**
Se o seu `.env` tem `MYSQL_USER=root`, a imagem oficial recusa iniciar (`root` é reservado — a
variável serve para criar um usuário comum). Não use `env_file` no serviço do banco; passe só
`MYSQL_ROOT_PASSWORD` e `MYSQL_DATABASE` por `environment`.

**Aparece um `.pnpm-store` dentro do projeto**
Falta o `npm_config_store_dir`. Sem ele, o pnpm cria o store ao lado do `node_modules`.

**`corepack: EACCES permission denied`**
Ele tenta escrever em `/usr/local/bin`, que é do root. O `doca` já contorna instalando os
atalhos no HOME — confirme que o `PATH` do `doca.yml` começa com `/home/node/.local/bin`.

**Estou dentro ou fora do container?**
O prompt mostra `usuário@projeto:/workspace`. Ou cheque `$DOCA_INSIDE` — ela só existe dentro.

---

## Licença

MIT.
