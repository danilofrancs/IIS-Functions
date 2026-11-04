# IIS - Recycle de pool por cliente

Playbook: `recycle-pool-cliente.yml`

Faz o **recycle das Application Pools de um cliente** (ex.: CLIENTEFARMA) sem afetar os
outros clientes. Você informa o nome, ele procura nos servidores IIS, mostra o que
vai fazer e só executa se você confirmar.
---

## Como ler o plano

```
[web03] sites: cliente.empresax.com.br
  VAI RECICLAR (6):
    + BIOLAB  ->  cliente.empresax.com.br/app, .../pdv
    + Biolab Ecommerce  ->  cliente.empresax.com.br/Ecommerce
  NAO VAI RECICLAR (11):
    - Root (Started): COMPARTILHADA com mais 15 site(s), ex.: uq..., arese...
    - BiolabHomolog ApiIntegracaoGam (Stopped): pool Stopped
```

| Motivo no plano  | O que significa                                                   |
|------------------|-------------------------------------------------------------------|
| `COMPARTILHADA`  | a pool atende outros clientes; reciclar derrubaria todos eles     |
| `PROTEGIDA`      | pool do sistema (DefaultAppPool, .NET v4.5...); nunca é reciclada |
| `pool Stopped`   | a pool está parada; o playbook não liga nem recicla               |
| `INEXISTENTE`    | o site aponta para uma pool que não existe                        |

---

## O que ele faz e o que NÃO faz

**Faz**
- Recicla só as pools **exclusivas** do cliente.
- Um servidor por vez, com 15 s de pausa entre eles.
- Confere se a pool voltou para `Started`; se não voltar em 60 s, **para tudo**.
- Grava cada recycle em `logs/iis-recycle-pool.log` (data, usuário, cliente, servidor, pool, status).

**Não faz**
- Não recicla pool compartilhada com outros clientes (ex.: `Root`, `asp`, `ExecPDV`).
- Não liga pool parada.
- Não altera nada sem `-e confirmar=SIM`.

O recycle do IIS é "suave": sobe um processo novo e o antigo termina as
requisições em andamento. Normalmente o site não cai.

---

## Opções

| Opção                                  | Para que serve                                         |
|----------------------------------------|--------------------------------------------------------|
| `-e cliente=NOME`                      | **obrigatório**; parte do nome do site/aplicação       |
| `-e confirmar=SIM`                     | executa de verdade (sem isso, só mostra o plano)       |
| `-e busca_exata=true`                  | nome exato em vez de "contém" (biolab ≠ biolabhomolog) |
| `-e pausa_entre_servidores=30`         | segundos entre um servidor e outro (padrão 15)         |
| `-e max_sites_por_servidor=2`          | limite da trava de sites (padrão 2)                    |
| `-e max_pools_por_servidor=10`         | limite da trava de pools (padrão 10)                   |
| `-e permitir_pool_compartilhada=true`  | ⚠️ recicla também pools compartilhadas: **derruba outros clientes**. Evite. |

---

## Pelo Jenkins

Arquivo: `Jenkinsfile.recycle-pool`

**Configurar o job (uma vez)**
1. Novo item → **Pipeline**.
2. *Pipeline script from SCM* → Git → repositório do GitHub, branch `main`.
3. *Script Path*: `Jenkinsfile.recycle-pool`
4. Salvar e rodar uma vez ("Build Now"): a 1ª execução só carrega os parâmetros.

**Usar:** *Build with Parameters*

| Parâmetro     | O que colocar                                          |
|---------------|--------------------------------------------------------|
| `CLIENTE`     | ex.: `biolab`                                          |
| `SERVIDOR`    | `webhomolog` (padrão), `web03`, ..., `PRODUCAO`, `TODOS` |
| `ACAO`        | `PLANO` = só mostra / `EXECUTAR` = recicla             |
| `BUSCA_EXATA` | marque para nome exato                                 |

Com `EXECUTAR`, o Jenkins mostra o plano e **para pedindo aprovação**
("Sim, reciclar"). Confira o plano no log antes de aprovar. Sem aprovação em
15 min, cancela sozinho.

**Credencial:** usa a credencial `iis-deploy-winrm` do Jenkins. O `vault.yml`
não é usado no Jenkins (nem precisa da senha do vault).

O log de cada recycle fica anexado ao build (*Artifacts* → `iis-recycle-pool.log`),
com o usuário do Jenkins que disparou.

---

## Atenção

- A **raiz** de alguns sites (ex.: BIOLAB no web03) usa a pool `Root`, compartilhada
  com vários clientes. O playbook **não** recicla essa pool. Se o problema estiver
  na raiz do site, o ideal é criar uma pool própria para o cliente no IIS.
- Na dúvida, teste antes no homolog: `--limit webhomolog`.
- **PID igual antes/depois no resultado é normal.** O processo antigo ainda está
  finalizando as requisições, e o novo `w3wp` só sobe na próxima requisição ao site.
  `-` nos PIDs = a pool estava ociosa (sem processo rodando). O que vale é o status `OK`.
