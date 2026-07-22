# CONDOR WMS — Separação de Pedidos

Aplicativo web (PWA) para separação de Ordens de Produção no almoxarifado, com Google Sheets como banco de dados via Google Apps Script.

## Arquivos entregues

| Arquivo | Função |
|---|---|
| `index.html` | Estrutura das telas (login, dashboard, OPs, separação, upload) |
| `style.css` | Identidade visual industrial (preto/laranja/branco) |
| `script.js` | Toda a lógica do app (SPA, API, gráficos, importação) |
| `manifest.json` | Configuração do PWA (instalar no tablet) |
| `service-worker.js` | Cache do app shell para abrir rápido offline |
| `Codigo.gs` | Backend — cole na aba **Extensões > Apps Script** da planilha |
| `CONDOR_WMS_modelo_planilha.xlsx` | Modelo pronto com as 4 abas e exemplos |
| `assets/logo.png` | Logo Condor usada no login e no menu lateral |

---

## Passo 1 — Criar a planilha Google Sheets

1. Faça upload do arquivo `CONDOR_WMS_modelo_planilha.xlsx` para o seu Google Drive e abra-o com o Google Sheets (ou **Arquivo > Importar** dentro de uma planilha em branco).
2. Confirme que existem 4 abas: `OP`, `ITENS_OP`, `USUARIOS`, `HISTORICO`.
3. Apague as linhas de exemplo (destacadas em amarelo) antes de usar em produção — mantenha apenas o cabeçalho.
4. Na aba `USUARIOS`, cadastre os operadores reais. **Importante:** o modelo original do briefing tinha as colunas `ID, NOME, LOGIN, PERFIL` — foi adicionada a coluna **SENHA**, necessária para a tela de login. Preencha um login e senha para cada usuário. `PERFIL` deve ser exatamente `Administrador` ou `Separador`.

## Passo 2 — Publicar o backend (Apps Script)

1. Na planilha, vá em **Extensões > Apps Script**.
2. Apague o conteúdo do arquivo `Código.gs` que abrir e cole todo o conteúdo de `Codigo.gs` (deste pacote).
3. Salve (ícone de disquete).
4. Clique em **Implantar > Nova implantação**.
5. Em "Selecionar tipo", clique na engrenagem e escolha **Aplicativo da Web**.
6. Configure:
   - **Executar como:** Eu (seu e-mail)
   - **Quem pode acessar:** Qualquer pessoa
7. Clique em **Implantar**, autorize as permissões solicitadas (é a sua própria planilha) e copie a **URL do aplicativo da Web** (termina em `/exec`).

> Sempre que você editar `Codigo.gs`, é preciso fazer **Implantar > Gerenciar implantações > ✏️ editar > Nova versão** para as mudanças entrarem em vigor na URL publicada.

## Passo 3 — Conectar o frontend ao backend

1. Abra `script.js`.
2. No topo do arquivo, troque a URL de exemplo pela URL copiada no passo anterior:

```js
const CONFIG = {
  APPS_SCRIPT_URL: "https://script.google.com/macros/s/AKfycb.../exec"
};
```

3. Salve o arquivo.

## Passo 4 — Publicar o frontend (GitHub Pages)

1. Crie um repositório novo no GitHub (ex.: `condor-wms`).
2. Envie todos os arquivos do frontend: `index.html`, `style.css`, `script.js`, `manifest.json`, `service-worker.js` e a pasta `assets/` com `logo.png`.
   *(Não é necessário subir `Codigo.gs` nem o `.xlsx` — eles ficam apenas no Google Sheets/Apps Script.)*
3. Vá em **Settings > Pages**, selecione a branch `main` e a pasta raiz (`/`).
4. Acesse a URL gerada (ex.: `https://seuusuario.github.io/condor-wms/`).

## Passo 5 — Instalar no tablet

No Chrome do tablet, abra a URL publicada e toque em **Adicionar à tela inicial** (ou use o ícone de instalação na barra de endereço). O app abrirá em tela cheia, sem barra do navegador, como um aplicativo nativo.

---

## Como funciona a API (resumo técnico)

O frontend faz `POST` para a URL do Apps Script com corpo `{ action, payload }` e `Content-Type: text/plain` (evita bloqueio de CORS no Apps Script). O backend responde sempre `{ ok: true, result }` ou `{ ok: false, error }`.

| Ação | Payload | O que faz |
|---|---|---|
| `login` | `{ login, senha }` | Autentica contra a aba USUARIOS |
| `getOPs` | — | Lista OPs com contagem de itens/separados |
| `getItensByOP` | `{ op }` | Lista itens de uma OP específica |
| `confirmarSeparacao` | `{ id, usuario }` | Marca item como separado (não permite desfazer) |
| `concluirOP` | `{ op, usuario }` | Fecha a OP se todos os itens já estiverem separados |
| `uploadOP` | `{ rows: [...] }` | Cria a OP (se não existir) e importa os itens |
| `getDashboard` | — | Retorna indicadores e dados para os gráficos |

Toda ação de separação, conclusão de OP ou importação é registrada na aba `HISTORICO` (auditoria), conforme exigido no briefing.

## Formato do arquivo de importação (upload de OP)

Excel (.xlsx) ou CSV com as colunas (a ordem não importa, os nomes são reconhecidos com ou sem acento):

```
OP | CLIENTE | ITEM | MICROSIGA | DESCRIÇÃO | ENDEREÇO | LOTE | QTD
```

O app agrupa as linhas por `OP`: se a OP ainda não existir na aba `OP`, ela é criada automaticamente com status `PENDENTE`; todos os itens são inseridos em `ITENS_OP` com `SEPARADO = 0`.

## Regras de negócio implementadas

- Um item, uma vez confirmado como separado, não pode ser desfeito pela interface nem sobrescrito pelo backend.
- Uma OP só pode ser concluída quando **100%** dos itens estiverem separados — o botão "Concluir OP" fica desabilitado até lá.
- Toda separação registra usuário e data/hora automaticamente (vindos da sessão logada, não digitados manualmente).
- Apenas usuários com `PERFIL = Administrador` veem o menu "Importar OP".
- Status da OP nas telas (Pendente / Em andamento / Concluído) é calculado dinamicamente a partir da proporção de itens separados.

## Personalização rápida

- **Cores:** variáveis `--orange`, `--black-*` no topo de `style.css`.
- **Logo:** substitua `assets/logo.png` mantendo o mesmo nome de arquivo.
- **Fuso horário das datas:** ajuste em `Codigo.gs`, função `fmtDate` (`Session.getScriptTimeZone()` já usa o fuso configurado na própria planilha — confira em **Configurações do projeto** no Apps Script ou em **Arquivo > Configurações** da planilha).
