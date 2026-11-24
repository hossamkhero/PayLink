# PayLink

Bill-payment and wallet backend for Egyptian utility bills: electricity, water, mobile top-up, internet.

Customers top up a digital wallet and pay bills through it; PayLink charges a service fee and pays commission to agents. The system's core obligation is correctness with money — every balance change is a ledger entry, and balances are derived, never overwritten.


## Tech stack

- Python 3 + FastAPI
- PostgreSQL 16
- Docker + docker compose
- GitHub Actions

## Quick start

```bash
git clone https://github.com/hossamkhero/PayLink && cd paylink
docker compose up
```

The app starts at http://localhost:8080. At Phase 1 it serves only GET /health, which returns {"status": "ok"}.

Repository layout
```text
docs/
  requirements.md          Requirements pack (Phase 1)
  design/                  Mermaid diagram sources
  prompt-log.md            AI prompt log
src/
  wallet/                  Accounts, wallet, ledger
  billers/                 Biller catalog
  payments/                Payment life cycle, orchestration
  integrations/            One adapter per biller
  fees/                    Fee and commission calculation
  notifications/           SMS / push / email with retry
  fraud/                   Chain-of-responsibility checks
  reconciliation/          Nightly settlement comparison
Dockerfile
docker-compose.yml
```

Money columns are DECIMAL(12,2) EGP in the database; conversion to piasters happens only inside the ElectroCo adapter.
