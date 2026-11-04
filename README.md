# IIS - Recycle de pool por cliente

Playbook: `recycle-pool-cliente.yml`

Faz o **recycle das Application Pools de um cliente** (ex.: BIOLAB) sem afetar os
outros clientes. Você informa o nome, ele procura nos servidores IIS, mostra o que
vai fazer e só executa se você confirmar.

> Rode sempre a partir da raiz do projeto (`ORQUESTRACAO/`).

---

## Passo a passo

**1. Ver o plano (não altera nada)**
```bash
ansible-playbook IIS/recycle-pool-cliente.yml -e cliente=biolab --limit web03 --ask-vault-pass
```

**2. Conferir a lista "VAI RECICLAR"** no resultado.
Só devem aparecer pools do próprio cliente. Se aparecer algo estranho, **pare**.

**3. Executar**
```bash
ansible-playbook IIS/recycle-pool-cliente.yml -e cliente=biolab -e confirmar=SIM --limit web03 --ask-vault-pass
```

O passo 1 e o passo 3 são o mesmo comando; a diferença é só o `-e confirmar=SIM`.

---

## Onde ele roda (`--limit`)

O `--limit` é um filtro: o playbook roda **somente** nos servidores listados.

| Comando                       | Onde recicla                                |
|-------------------------------|---------------------------------------------|
| `--limit web03`               | só no web03                                 |
| `--limit webhomolog`          | só no homolog                               |
| `--limit web03,webhomolog`    | nos dois                                    |
| sem `--limit`                 | em todo servidor IIS onde achar o cliente   |

Para conferir os servidores sem executar nada, acrescente `--list-hosts`.

---

## Como ler o plano

```
[web03] sites: BIOLAB.salesfarma.com.br
  VAI RECICLAR (6):
    + BIOLAB  ->  BIOLAB.salesfarma.com.br/app, .../pdv
    + Biolab Ecommerce  ->  BIOLAB.salesfarma.com.br/Ecommerce
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

## Travas de segurança

| Trava                                         | Resultado                         |
|-----------------------------------------------|-----------------------------------|
| Nome com menos de 3 letras ou caractere estranho | não roda                       |
| Cliente não encontrado                        | não roda                          |
| Busca casou com mais de 2 sites num servidor  | não roda (nome genérico demais)   |
| Mais de 10 pools a reciclar num servidor      | não roda                          |
| Nenhuma pool elegível                         | não roda                          |

Quando uma trava bloqueia, **nada é alterado**.

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

Arquivo: `IIS/Jenkinsfile.recycle-pool`

**Configurar o job (uma vez)**
1. Novo item → **Pipeline**.
2. *Pipeline script from SCM* → Git → repositório do GitHub, branch `main`.
3. *Script Path*: `IIS/Jenkinsfile.recycle-pool`
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

**Agente:** precisa de um agente Linux com label `ansible` que tenha
`ansible` e `python3-winrm`. Se o seu label for outro, troque no início do Jenkinsfile.

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
