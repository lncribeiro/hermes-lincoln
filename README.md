# Hermes Agent na VPS com Docker e Caddy

Execute o [Hermes Agent](https://hermes-agent.nousresearch.com/) usando a imagem
oficial da Nous Research. Cada instância é um projeto Docker Compose com seu
próprio volume para configuração, chaves, sessões e arquivos em `/opt/data`.
O Caddy que já atende a VPS também termina o HTTPS dos novos subdomínios.
A API do gateway fica desativada por padrão.

## Primeira instância

Na VPS, clone este repositório e entre na pasta. São necessários Docker Engine
com o plugin Compose, um subdomínio apontando para a VPS e acesso à configuração
do Caddy existente.

```sh
install -m 600 .env.example .env.lincoln
nano .env.lincoln
```

Defina `HERMES_PUBLIC_URL` com o endereço HTTPS real, por exemplo
`https://hermes.seudominio.com.br`. Escolha uma senha forte e exclusiva para
`HERMES_DASHBOARD_PASSWORD`. Você pode gerar uma com `openssl rand -hex 32`.
Essa senha é a do painel. Configure as credenciais do provedor de IA no
assistente do Hermes. Se usar caracteres especiais como `$` na senha,
coloque o valor entre aspas simples no arquivo de configuração.

Escolha o arquivo complementar de acordo com a instalação do Caddy e mantenha
o mesmo valor de `COMPOSE_FILE` nos comandos seguintes.

- **Caddy instalado no host.** Use
  `export COMPOSE_FILE=compose.yaml:compose.caddy-host.yaml`.
  O Hermes publica o painel apenas em `127.0.0.1`, na porta definida por
  `HERMES_HOST_PORT` (por exemplo, `19119`).
- **Caddy em Docker.** Use
  `export COMPOSE_FILE=compose.yaml:compose.caddy-docker.yaml`.
  Defina `CADDY_NETWORK` com o nome de uma rede Docker existente à qual o
  contêiner Caddy está conectado. O upstream é
  `<HERMES_INSTANCE>-dashboard:9119` e o Hermes dispensa portas no host.

```sh
docker compose --env-file .env.lincoln pull
docker compose --env-file .env.lincoln up -d
docker compose --env-file .env.lincoln exec hermes hermes setup
docker compose --env-file .env.lincoln restart hermes
docker compose --env-file .env.lincoln ps
```

Se o painel ainda estiver iniciando, acompanhe com
`docker compose --env-file .env.lincoln logs --tail=80 hermes`.
O estado `healthy` verifica a disponibilidade HTTP do painel. Para validar
o modelo configurado, envie uma mensagem pelo painel após fazer login.

## Conectar uma assinatura do ChatGPT ou do Claude

Depois de subir a instância, execute o login do provedor desejado na VPS.
Essas duas opções usam OAuth da assinatura elegível, com os limites e modelos
disponíveis na conta. O login do provedor é separado da senha do painel.

**ChatGPT pelo OpenAI Codex**

```sh
docker compose --env-file .env.lincoln exec hermes \
  hermes auth add openai-codex --type oauth --no-browser
```

Abra no seu computador o endereço exibido no terminal, entre na conta do
ChatGPT e informe o código de dispositivo. Se a OpenAI pedir para habilitar
o login por código de dispositivo, use as configurações de segurança da conta.

**Claude Pro ou Max**

```sh
docker compose --env-file .env.lincoln exec hermes \
  hermes auth add anthropic --type oauth --no-browser
```

Abra o link exibido no terminal, autorize com a conta do Claude e cole o
código de autorização de volta no próprio terminal.

**Selecionar o modelo**

```sh
docker compose --env-file .env.lincoln exec hermes hermes model
docker compose --env-file .env.lincoln restart hermes
```

No seletor, escolha OpenAI Codex ou Anthropic de acordo com o login que
você fez. As credenciais ficam no volume da instância e sobrevivem à
recriação do contêiner. Para uma segunda instância, execute o login com o
arquivo dela. Depois, envie uma mensagem pelo painel para verificar se a
conta tem acesso ao modelo escolhido.

## Adicionar ao Caddy

Inclua um bloco para cada subdomínio na configuração existente. Use
[Caddyfile.example](Caddyfile.example) como referência. Por exemplo, com
Caddy no host:

```caddyfile
hermes.seudominio.com.br {
    reverse_proxy 127.0.0.1:19119
}
```

Com Caddy em Docker, use `reverse_proxy hermes-lincoln-dashboard:9119`
no mesmo bloco. O Caddy encaminha também conexões WebSocket do painel.

Para que o Hermes reconheça o HTTPS terminado pelo Caddy e marque seus cookies
de sessão como `Secure`, configure `dashboard.trusted_proxies` com o IP do
proxy imediato **visto pelo contêiner Hermes**. No Docker em Linux, costuma
ser o IP do Caddy na rede compartilhada ou, para Caddy no host, o IP do
gateway da rede do projeto. Confirme os endereços com `docker network inspect`
e use um IP específico (ou um CIDR restrito aos proxies que você administra).
Por exemplo, se o proxy imediato chega como `172.30.0.2`:

```sh
docker compose --env-file .env.lincoln exec hermes \
  hermes config set dashboard.trusted_proxies '["172.30.0.2"]'
docker compose --env-file .env.lincoln restart hermes
```

Valide e recarregue o arquivo ativo do Caddy. Para uma instalação no host
gerenciada pelo systemd com o caminho padrão:

```sh
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

Se estiver em Docker, execute `caddy validate` e `caddy reload` dentro
do contêiner Caddy com o caminho do Caddyfile montado. Abra o subdomínio
em HTTPS e entre com o usuário e a senha da instância.

## Adicionar outra instância

Crie outro arquivo a partir de `.env.example`, por exemplo `.env.equipe`.
Dê a ele um `HERMES_INSTANCE` exclusivo, como `hermes-equipe`, outro
subdomínio e outra senha. Com Caddy no host, escolha também uma porta livre
diferente, como `19120`. Repita os comandos com `--env-file .env.equipe`
e adicione seu bloco ao Caddy. Cada instância usa seu próprio volume
`<HERMES_INSTANCE>_data`. Se conectar bots a canais de mensagens, use os
tokens próprios de cada instalação.

## Operação e atualização

Use `docker compose --env-file .env.lincoln stop` para pausar a primeira
instância e `up -d` para iniciá-la novamente. Use o mesmo `COMPOSE_FILE`
de quando a criou. O comando `down -v` apaga o volume e os dados.

A imagem é fixada por digest e tem variantes para Linux amd64 e arm64.
Antes de atualizar, faça backup do volume. Defina `HERMES_IMAGE` no
arquivo da instância com a nova referência oficial, de preferência por
digest, e execute `pull` e `up -d` novamente.
