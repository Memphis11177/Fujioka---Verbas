# Cadastro de Verbas — Fujioka

Aplicação para cadastrar e acompanhar verbas comerciais por **linha comercial** e **marca**.
Nesta etapa o foco é o **cadastro**; a integração com o sistema da empresa (que vai trazer os
créditos e demais informações) entra numa etapa futura e já tem lugar reservado na estrutura.

## Como abrir

Dê dois cliques em `index.html`. Não precisa de instalação nem de servidor.
Na primeira abertura o sistema carrega **dados de exemplo** para a apresentação.
Para começar do zero, use **Base de dados → Apagar todos os dados**; para voltar à demonstração,
**Restaurar dados de exemplo**.

## Publicar na Netlify

O projeto é um site estático (sem build). O `netlify.toml` na raiz já configura tudo.

**Opção 1 — arrastar e soltar (mais rápido)**
1. Acesse https://app.netlify.com/drop e entre na sua conta.
2. Arraste a pasta **Projeto Fujioka** inteira para a área indicada.
3. A Netlify gera um endereço (ex.: `https://nome-aleatorio.netlify.app`). Em
   *Site configuration → Change site name* dá para trocar o nome.
4. Para atualizar depois: no site, aba *Deploys*, arraste a pasta de novo.

**Opção 2 — pelo GitHub (atualiza sozinho a cada alteração)**
1. Envie a pasta para um repositório no GitHub.
2. Na Netlify: *Add new site → Import an existing project → GitHub* e escolha o repositório.
3. Deixe *Build command* vazio e *Publish directory* como `.` (já vem do `netlify.toml`).

> Os dados ficam salvos **no navegador de quem acessa** (localStorage). Cada pessoa/computador
> vê sua própria base, começando pelos dados de exemplo. Para compartilhar dados entre usuários
> será preciso a integração com o backend (ver "Integração").

## O que dá para fazer

| Aba | Função |
| --- | --- |
| Resumo | Verba prevista, débitos, créditos e saldo por linha → marca → lançamentos |
| Lançamento de Verbas | Cadastrar, editar, excluir e filtrar lançamentos (Verba Gross, imposto, Verba Net, status) |
| Tipos de verba | Cadastrar, renomear e excluir tipos (Sell-out, Rebate, etc.) |
| Linhas e marcas | Cadastrar linhas comerciais e suas marcas, com verba prevista e cor |
| Base de dados | Exportar/importar a base em JSON, restaurar exemplo, apagar tudo |

### Tipos de verba e campos do lançamento

Todo lançamento tem **Data Início, Data Final e Marca** (além de linha, status, descrição e valores).
O tipo de verba escolhido define os campos extras:

| Tipo | Campos extras |
| --- | --- |
| Sell Out | Itens (com Data Início, Data Final e Marca em cada item): Agrupador, Código, Descrição, Qtd. Liberada, Valor Unitário, Valor Total, Valor Net, Qtd. Realizada, Change |
| Sell In (Pedido) | Itens: Código, Descrição, Qtd. Disponível, Qtd. Atendida |
| Bônus, Contrato, Marketing, DMS, Pagamento de Campanha | — (período e marca) |

- Sell Out: **Valor Total** = Qtd. Liberada × Valor Unitário; **Valor Net** = Valor Total − imposto;
  a Verba Gross do lançamento é a soma dos Valores Totais.
- Os campos de cada formulário ficam em `js/config.js` → `regras.modelos`. Um tipo novo pode usar
  qualquer um dos formulários (escolhido na aba Tipos de verba).

### Regras de negócio implementadas

- **Valor do imposto** = Verba Gross × Perc. imposto (padrão 9,25%).
- **Verba Net** = Verba Gross − Valor do imposto.
- Lançamentos manuais são **débitos**. **Créditos** virão da integração (nos dados de exemplo
  aparecem alguns créditos simulados marcados como "Origem: sistema", que não podem ser editados).
- Ações com status **Cancelada** não entram no saldo.
- **Saldo** = créditos − débitos. **Disponível** (por marca) = verba prevista − débitos.
- Não é possível excluir linha, marca ou tipo que já tenha lançamentos.
- O código do lançamento (`LV-0001`, `LV-0002`…) é gerado automaticamente.

## Estrutura

```
netlify.toml                   configuração de publicação na Netlify
index.html                     telas (HTML)
css/estilos.css                visual (tema preto/vermelho Fujioka)
js/config.js                   configuração: fonte de dados, URL da API, regras (imposto, status)
js/util.js                     formatação e cálculo da verba (sem dependência de tela)
js/dados/dados-exemplo.js      dados da demonstração
js/dados/repositorio-local.js  grava no navegador (localStorage) — usado hoje
js/dados/repositorio-api.js    grava no backend via HTTP — pronto para a integração
js/dados/repositorio.js        escolhe a fonte conforme config.js e documenta o contrato
js/app.js                      lógica das telas (só fala com o repositório)
```

A tela nunca acessa `localStorage` ou a rede diretamente: ela chama `Verbas.repositorio`,
que tem a mesma interface nos dois modos. Por isso a integração não exige mudar as telas.

## Integração (etapa futura)

1. Criar o backend que expõe os endpoints listados no topo de `js/dados/repositorio-api.js`
   (`/linhas`, `/marcas`, `/tipos`, `/lancamentos`, `/base`), em JSON, com os mesmos campos.
2. Em `js/config.js`, trocar `fonteDados: "local"` por `fonteDados: "api"` e ajustar `api.urlBase`
   (e `api.cabecalhos`, se houver autenticação).
3. Créditos vindos do sistema devem ser enviados como lançamentos com `natureza: "C"`,
   `origem: "integracao"` e `idExterno` (identificador no sistema de origem).
4. Para levar os dados já cadastrados na demonstração, use **Base de dados → Baixar base** e
   carregue o JSON no backend.

### Modelo de dados

```js
linha       { id, nome }
marca       { id, linhaId, nome, verbaPrevista, cor }
tipo        { id, nome, modelo /* "padrao" | "sellout" | "sellin" */ }
lancamento  { id, codigo, data /* AAAA-MM-DD */, dataInicio, dataFim, linhaId, marcaId, tipoId,
              itens /* [] ou linhas de Sell Out / Sell In, ver config.js */,
              natureza /* "D" débito | "C" crédito */,
              valorBruto, percImposto, valorImposto, valorLiquido,
              codigoDescricao, descricao,
              status /* Planejada | Em andamento | Concluída | Cancelada */,
              origem /* "manual" | "integracao" */, idExterno }
```

Bases exportadas pela versão anterior (`Linhas comerciais e marcas.html`) são convertidas
automaticamente ao importar.
