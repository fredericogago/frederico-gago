+++
title = "Reconciliação Automática de Contribuições (ERP ↔ Portal) com Python"
date = 2025-09-11T15:47:39+01:00
draft = false
description = "Padrões práticos para alinhar duas fontes de dados mensais — com tolerância monetária, concorrência limitada, retries com backoff e upsert idempotente."
tags = ["Python", "Automação", "Contabilidade Digital", "FastAPI", "Playwright", "Clean Architecture"]
categories = ["Engineering", "Automation"]
slug = "reconciliacao-contribuicoes-erp-portal-python"

[cover]
image = "/images/cover/reconcile-cover.png"
alt = "Ilustração de reconciliação de dados"
+++

> **Resumo:** Como automatizei a reconciliação mensal de contribuições entre um ERP e um portal externo.  
> Mostro **padrões reutilizáveis**: *bounded concurrency*, *retry* com backoff + jitter, **tolerância monetária**, **bucketização por período+taxa** e **upsert idempotente** de divergências — usando *value objects* (`YearMonth`, `Money`) e *ports* (interfaces).  
> **Sem expor integrações reais**: o foco é a arquitetura e as ideias reutilizáveis.

---

## ⚙️ Contexto & Problema

Em RH, após o processamento salarial (até dia 10 em PT), é enviado o mapa de contribuições do mês anterior para o portal oficial.  
Se ocorrerem **alterações** após o envio (ajustes, faltas, prémios, correções), é **fácil esquecer** o reenvio do mapa corrigido — gerando **inconsistências** com risco regulatório e retrabalho.

**Objetivo:** conferir **mensalmente** se o que está no ERP **bate** com o portal externo, sinalizando divergências de forma **automática**, rápida e auditável.

- Tempo poupado médio (por empresa/mês): ~**5 min** (quando existia política de conferência manual).  
- Redução de falhas humanas: **eliminação** de esquecimentos sistemáticos.  
- Base para **alertas** e **relatórios** automáticos.

> Nota: este artigo não é aconselhamento jurídico; o foco é reduzir risco operacional e facilitar conformidade.

---

## 🧱 Padrões Arquiteturais (reutilizáveis)

- **Value Objects**
  - `YearMonth` (faixas, ordenação, últimos 12 meses concluídos…)
  - `Money` (moeda, aritmética segura, formatação PT)
- **Bucketização** por `(ano, mês, taxa)` antes de comparar.
- **Tolerância monetária** para evitar falsos positivos por arredondamentos.
- **Concorrência limitada** (*bounded semaphores*) para proteger recursos (DB/ERP/Browser).
- **Retry** com backoff exponencial + jitter para chamadas mais frágeis (portal).
- **Upsert idempotente** de divergências (grava/atualiza só se mudou).
- **Ports/Adapters**: `IErpService`, `IPortalService`, `IRepo`.

---

## 🧩 Value Objects (excerto)

**`YearMonth`** (intervalos de 12 meses concluídos, ordenação, formatação):

```python
@dataclass(frozen=True)
class YearMonth:
    year: int
    month: int
    fmt: str = field(default="%Y-%m", repr=False)
    # ... validação, formatted, parse, range, comparação, last_12_completed_months() ...
```

**`Money`** (inteiros em cêntimos, aritmética segura, formatação PT):

```python
@dataclass(frozen=True)
class Money:
    amount_cents: int
    original: str
    input_value: Any
    currency: CurrencyCode = CurrencyCode.EUR
    # .amount -> Decimal, from_raw(), +, -, *, /, format_pt(), parse_percentage() ...
```

Estes VOs isolam complexidade de datas mensais e quantias monetárias (arredondamentos, parsing, igualdade), reduzindo bugs na reconciliação.

---

# 🔌 Portas (interfaces) & DTOs

# Divergência (DTO) — “fotografia” estável p/ persistir e reportar

```python
@dataclass(frozen=True)
class MismatchDTO:
    company: CompanyRef
    period: YearMonth
    erp_remuneration: Money
    portal_remuneration: Money
    erp_contribution: Money
    portal_contribution: Money
    difference: Money
    tax: Decimal
```
# Repositório (abstrato) — garante upsert idempotente e consultas

```python
class IRepo(..., ABC):
    @abstractmethod
    async def upsert_contribution_if_changed(self, entity: Contribution) -> tuple[Contribution, bool]: ...
    @abstractmethod
    async def get_contributions_that_are_not_yet_checked_with_relationship(self, **filters) -> list[Contribution]: ...

    # ... outros métodos ...
```

# ERP (agrega por período)

```python
@dataclass(frozen=True, slots=True)
class ContributionByPeriod:
    period: YearMonth
    type_of_contribution: int
    total_rem: Money
    tax_employee: Decimal
    tax_employer: Decimal
    total_contribution: Money
```

# Portal (agrega por taxa)

```python
@dataclass(frozen=True, slots=True)
class ContributionByTax:
    period: YearMonth
    remuneration: Money
    tax: Decimal
    contribution: Money
```

---

# 🧠 Algoritmo (alto nível)

1. **Janela temporal:** últimos 12 meses concluídos (`YearMonth.last_12_completed_months()`).
2. **Obter dados:**
   * ERP: `ContributionByPeriod` (taxa = empregado + entidade)
   * Portal: `ContributionByTax`
3. **Bucketizar** por `(ano, mês, taxa)`:
   * ERP: soma de `total_rem` e `total_contribution`
   * Portal: soma de `remuneration` e `contribution`
4. **Comparar com tolerância:**
   * Se `|erp_rem - portal_rem| <= tol` e `|erp_con - portal_con| <= tol` → OK
   * Caso contrário → **MismatchDTO**
5. **Upsert idempotente:** gravar/atualizar divergências só se mudaram.
6. **Opcional:** emitir eventos (relatório de RH, erros, métricas).

# 🧵 Concorrência & Retry (exemplo simplificado)

```python
class ReconcileService:
    def __init__(..., max_portal=2, max_erp=1, attempts=3, base=1.0, jitter=0.3, tol_eur=Decimal("0.50")):
        self._portal_sem = asyncio.BoundedSemaphore(max_portal)
        self._erp_sem    = asyncio.BoundedSemaphore(max_erp)
        self._attempts, self._base, self._jitter = attempts, base, jitter
        self._tol = tol_eur

    async def _portal_with_retry(self, portal, start, end):
        async with self._portal_sem:
            attempt = 0
            while True:
                try:
                    return await portal.get_contributions_by_tax(start, end)
                except Exception:
                    attempt += 1
                    if attempt >= self._attempts:
                        raise
                    delay = self._base * (2 ** (attempt - 1)) + random.uniform(0, self._jitter)
                    await asyncio.sleep(delay)

    def _close(self, a: Money, b: Money) -> bool:
        return abs(a.amount - b.amount) <= self._tol
```
    
**Porque assim?** 
* Semáforos evitam saturar BD/ERP/sessões de browser.
* Retry com backoff + jitter suaviza “picos” e lida melhor com limitações/captcha.
* Tolerância monetária reduz falsos positivos por arredondamentos.

# 📊 Exemplo de Comparação (core da decisão)

```python
def compare(company: CompanyRef, period: YearMonth, tax: Decimal,
            erp_rem: Money, erp_con: Money, por_rem: Money, por_con: Money,
            tol: Decimal) -> MismatchDTO | None:

    rem_close = abs(erp_rem.amount - por_rem.amount) <= tol
    con_close = abs(erp_con.amount - por_con.amount) <= tol

    if rem_close and con_close:
        return None

    return MismatchDTO(
        company=company, period=period, tax=tax,
        erp_remuneration=erp_rem, portal_remuneration=por_rem,
        erp_contribution=erp_con, portal_contribution=por_con,
        difference=(por_con - erp_con),
    )
```

---

# ✅ Resultados (exemplo realista)

* **Tempo:** ~5 min poupados por empresa/mês quando havia conferência manual.
* **Qualidade:** menos esquecimentos de reenvio; divergências explícitas e auditáveis.
* **Escalabilidade:** processa várias empresas em paralelo (concorrência limitada).
* **Extensível:** serve de padrão para outras reconciliações (IVA, retenções, bancos).

    
    Métricas exatas variam por contexto; percentagens e tempos acima são indicativos.

---

# 🔒 Privacidade & Segurança

* Não exponho endpoints nem selectors do portal/ERP.
* Exemplos usam **dados fictícios e adapters** genéricos.
* O “core” de integrações (autenticação, Playwright, queries) permanece privado.

---

# 🛠️ Roadmap (ideias futuras)

* Painel com **alertas** por empresa/período.
* **Auto-retry** programado e digest semanal de divergências.
* Geração de relatórios **PDF/Excel** para auditoria.
* Integração com **fila de tarefas** (ex.: Celery/RQ) e notificações (e-mail/Teams).

---

# 📦 Como reutilizar os padrões

* Leva os **Value Objects** YearMonth e Money para o teu domínio.
* Cria ports (`IErpService`, `IPortalService`, `IRepo`) e implementa adapters locais.
* Aplica bucketização + tolerância antes de comparar.
* Usa semáforos separados por recurso e retry com backoff+jitter para o alvo mais instável.
* Upsert idempotente em repositório para não duplicar divergências.

---

# 💬 Fala comigo

Trabalhas em contabilidade/fintech e queres explorar **automação de reconciliações** (SAF-T, contribuições, retenções, bancos, etc.)?

Adoro transformar processos repetitivos em **sistemas robustos e auditáveis.**

* [LinkedIn](https://www.linkedin.com/in/frederico-gago-5849281aa)
* [GitHub](https://github.com/fredericogago)

