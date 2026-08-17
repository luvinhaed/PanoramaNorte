# Painel Operacional - Norte

Site estático (GitHub Pages) que exibe o painel, e um script Python
(`scripts/updater.py`) que roda **local, o tempo todo, na sua máquina/servidor**,
busca os dados nas APIs, monta o `data/dashboard_data.json` e publica esse
arquivo direto no repositório do GitHub via API (Contents API), sem precisar
de `git` instalado nem de push por SSH.

```
index.html          página do painel
app.js               lê data/dashboard_data.json e desenha tudo
style.css
assets/car.png
config/
  app_config.json    título, URLs das APIs, intervalo de atualização (front)
data/
  dashboard_data.json  gerado pelo updater — é o que o site lê
  dashboard_data.js    mesmo conteúdo, versão "window.DASHBOARD_DATA = {...}"
                        (fallback pra quando o index.html é aberto direto
                        como arquivo, protocolo file://)
  metadata.json         resuminho da última atualização (hash, contagens)
  territorio.json       município -> regional (fallback do mapa/tabela)
  regiao.geojson         polígonos dos municípios (mapa)
  NORTE.xlsx             se existir, é a fonte de verdade do territorio.json
                          (o updater recria o .json a partir dele a cada rodada)
scripts/
  updater.py           roda pra sempre, atualiza os dados e publica no GitHub
  serve.py             servidor local só pra pré-visualizar o site
  requirements.txt
```

## 1. Subir o site no GitHub Pages

1. Crie um repositório no GitHub (pode ser público ou privado — Pages
   funciona nos dois, privado só exige plano pago pra Pages).
2. Suba todo o conteúdo desta pasta pra branch `main`.
3. Em **Settings → Pages**, selecione a branch `main` e a pasta `/ (root)`.
4. Pronto — o site fica em `https://SEU-USUARIO.github.io/SEU-REPO/`.

Esse repositório é só o que o navegador serve. Ele **não** roda nada:
quem atualiza `data/dashboard_data.json` é o `updater.py` rodando na sua
máquina, que empurra o arquivo novo pra essa mesma branch a cada ciclo.

## 2. Configurar e rodar o updater localmente

Abra `scripts/updater.py` e edite o topo do arquivo:

```python
GITHUB_TOKEN = "COLOQUE_SEU_TOKEN_AQUI"   # fine-grained token, permissão "Contents: Read and write", só nesse repo
GITHUB_OWNER = "seu-usuario-ou-org"
GITHUB_REPO  = "painel-norte"
GITHUB_BRANCH = "main"
```

Não tem variável de ambiente de propósito — o script foi pensado pra ficar
rodando pra sempre num PC/servidor que só você acessa. **Não suba esse
arquivo já preenchido com o token pra um repositório público.** Se o
repositório do painel for público, guarde o `updater.py` preenchido fora
dele (ou num repo privado à parte) e copie só o `dashboard_data.json`/`.js`
gerado pro repo público.

Instalar dependência e rodar:

```bash
cd scripts
python -m venv .venv
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # Linux/Mac
pip install -r requirements.txt

python updater.py             # loop contínuo — roda pra sempre, Ctrl+C pra parar
```

Outras formas de rodar:

```bash
python updater.py --once                 # um ciclo só e encerra
python updater.py --no-push               # atualiza só os arquivos locais em data/, não publica no GitHub
python updater.py --main-source api.json --legacy-source legado.json --no-push   # testa com arquivo local, sem bater na API
```

O intervalo do loop é o `refresh_seconds` do `config/app_config.json`
(padrão 180s), ou `--interval SEGUNDOS` pra sobrescrever.

## 3. Pré-visualizar o site localmente

```bash
python scripts/serve.py
```

Abre em `http://localhost:8000`. Ou simplesmente abra `index.html` direto
no navegador — nesse caso ele usa `data/dashboard_data.js` como fallback
(porque `file://` não deixa dar `fetch` no `.json`).

## Regras de classificação por Polo

Ordem de prioridade pra decidir o polo (INHUÇU / CAMOCIM / TIANGUÁ / SOBRAL)
de cada incidência:

1. **Alimentador** (API principal) / **Circuito** (API legada) — tabela fixa
   de códigos no topo do `updater.py`.
2. **Conjunto** (API principal), se o alimentador não bater ou não mapear.
3. **Sucursal**, só como último fallback, e só pra `SOBRAL`, `CAMOCIM` e
   `SÃO BENEDITO` (que não é um polo oficial — cai classificada como
   `INHUÇU`).

Se nenhuma das três bater, a incidência é descartada do painel (contada em
`metadata.mapped_by_alimentador/conjunto/sucursal` e `dropped_unmapped` no
log do updater).

O **mapa** continua funcionando só por coordenada geográfica (ponto dentro
do polígono do município em `regiao.geojson`), igual antes — essa
classificação por alimentador/conjunto/sucursal é usada pro resumo por
polo, rankings e gráfico de equipes, não pro mapa.

**Equipes** são classificadas por polo olhando o prefixo do nome/lead da
equipe (tabela `TEAM_PREFIX_POLO` no `updater.py`).

`Finalizado` é sempre excluído do painel.
