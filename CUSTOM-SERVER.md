# Servidor de compilação próprio para Starling 0.9.0

Este fork serve os mods nativos usando **o Starling oficial instalado no Android**. Não é necessário compilar outro APK ou modificar `jssantogit/starling` para usar a tela **Mods > Compile > Build server...**.

## Estado

- O Worker Cloudflare já está configurado para responder por `jssantogit/mod-builds`.
- `versions.json` inclui `0.9.0` fixado no commit oficial `ff303502a4f7982551a0e0d3622d9268efe9e829` (`v0.9.0`).
- O deploy Cloudflare, a autorização do GitHub e a primeira compilação **ainda precisam ser executados e validados**.
- A compatibilidade dos antigos portes BadOptimizations e Greedy Meshing com 0.9.0 **não está comprovada**.

## 1. Habilitar GitHub Actions

No fork `jssantogit/mod-builds`, confira **Settings > Actions > General** para permitir executar workflows. O workflow `base.yml` precisa salvar o cache da versão antes do primeiro build.

Em **Actions > base > Run workflow**, selecione a branch `main` (após integrar a PR), e informe:

- `version`: `0.9.0`
- `tools`: `starling` (branch oficial com scripts de compilação)

Aguarde a execução terminar com sucesso. Não dispare mods antes de a base existir: `build.yml` usa `actions/cache/restore` com `fail-on-cache-miss`.

## 2. Autorizar GitHub no Worker Cloudflare

Opção simples: criar um **fine-grained personal access token** com acesso **somente** a `jssantogit/mod-builds`, permissões do repositório:

- **Actions: Read and write** (disparar `build.yml` e ler execuções);
- **Contents: Read and write** (criar releases temporárias e guardar os resultados).

A alternativa mais robusta é uma GitHub App restrita ao repositório, com as mesmas permissões. Não salve nenhum token, PEM ou segredo no GitHub, no repositório ou na conversa.

## 3. Publicar o Worker (Cloudflare)

Em um computador com Node.js, usando terminal autenticado **na conta Cloudflare de quem controla o servidor**:

```bash
git clone https://github.com/jssantogit/mod-builds.git
cd mod-builds/worker
npm ci
npm test
npx wrangler login
npx wrangler secret put GITHUB_TOKEN
npx wrangler deploy
```

No comando `secret put`, cole o token somente quando solicitado pelo Wrangler. É armazenado como segredo no Cloudflare, não como variável pública. Se preferir GitHub App, use os segredos `GITHUB_APP_ID` e `GITHUB_APP_KEY` e o procedimento original do `wrangler.toml`.

O Wrangler mostrará uma URL HTTPS do tipo `https://starling-compile-jssantogit.<sua-conta>.workers.dev`. **Use a URL real apresentada**, não este exemplo e não a URL do repositório.

Se estiver testando uma PR ainda não mesclada, certifique-se de que os arquivos publicados e o `versions.json` da branch `main` correspondem à configuração testada.

## 4. Conectar o Starling

No Starling 0.9.0 oficial, abra **Mods > Compile > Build server...**, cole a URL HTTPS do Worker Cloudflare e toque em **Use this server**.

Comece por um mod de teste compatível com a base selecionada. O fluxo `/v1/build` cria uma biblioteca nativa de substituição do motor usando cache da versão; `/v2/mod` só serve determinados mods compatíveis com o SDK de biblioteca separada, não os bundles antigos de mixins.

## 5. Verificação e diagnóstico

1. Abra `https://github.com/jssantogit/mod-builds/actions` e confirme que `base 0.9.0` terminou com sucesso.
2. Ao solicitar compilação no aplicativo, verifique a execução do workflow `build`.
3. Confira o status de `requests` e `results` nas Releases do fork.
4. Se falhar, consulte os logs públicos no GitHub; **não compartilhe tokens ou chaves**.
5. Para portas antigas, primeiro valide os contratos de hooks no tag `v0.9.0`; o servidor por si só não garante compatibilidade.

## Segurança

O serviço compila código Rust de terceiros, e a biblioteca resultante recebe acesso nativo ao processo do jogo. Use apenas mods confiáveis e não encaminhe builds de usuários desconhecidos sem avaliação. A arquitetura original isola a etapa de compilação em sandbox e publica resultados por outra etapa; preserve essa separação. O endpoint do Worker é público caso seu endereço seja compartilhado, e jobs GitHub consomem a cota da conta.
