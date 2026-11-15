# Entrega — Onfly: consolidador de reservas de hotel

## Defeitos corrigidos

| # | Arquivo | Defeito | Causa raiz (1 frase) |
|---|---------|---------|----------------------|
| 1 | `repositories/csv_reservation_repository.py` | Só 200 das 300 reservas eram lidas | Um `break` ao atingir 200 linhas truncava o arquivo silenciosamente. |
| 2 | `services/currency_converter.py` | Valores em USD/EUR vinham muito menores e com imprecisão | A conversão **dividia** por `rate_to_brl` e passava por `float`, em vez de `amount * rate` em `Decimal`. |
| 3 | `services/status_policy.py` | `NO_SHOW` entrava em noites, valores, `by_month` e `top_hotels` | `is_billable` era `status != "CANCELLED"`, em vez de `status == "CONFIRMED"`. |
| 4 | `services/stay_calculator.py` | Noites erradas (e negativas) em estadias que viram o mês | Subtraía `checkout.day - checkin.day` em vez de `(checkout - checkin).days`. |
| 5 | `services/report_builder.py` | `top_hotels` vinha do menor para o maior valor | A ordenação usava `amount` crescente, sem inverter o sinal; o desempate por `hotel_id` ascendente já estava correto. |

## Como investiguei

1. Li o README (regras de negócio e amostra de validação) e todo o código em `src/`, comparando cada regra com a implementação, uma por uma.
2. Os testes fornecidos apontavam 5 áreas (câmbio, noites, status, ranking, regressão); revisando cada módulo, encontrei também o limite de 200 linhas, que **nenhum teste fornecido cobre**. Só aparece ao notar que o CSV tem 300 reservas.
3. Validei de forma independente: recalculei o dataset real com um script separado (`Decimal`, `date`), sem usar o código do projeto. Resultado idêntico: 1347 noites e total exato de `396169.9994`, que arredonda para `396170.00`.
4. Adicionei `tests/test_extra_regressions.py` (arquivo novo; os testes fornecidos não foram alterados) cobrindo: arquivo com mais de 200 linhas, desempate do ranking, exclusão de `CANCELLED`/`NO_SHOW` em `by_month`/`top_hotels`, virada de ano com vários quartos, soma exata com arredondamento half-up só na serialização e média `0.00` sem noites.

## Como validar

```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
pytest -q

PYTHONPATH=src python -m onfly_report --input data/exemplo-validacao.csv \
  --rates data/taxas_cambio.csv --output output/exemplo.json
PYTHONPATH=src python -m onfly_report --input data/reservas_hoteis.csv \
  --rates data/taxas_cambio.csv --output output/report.json
```

## Resultado

- Amostra de validação idêntica à tabela do README (12 / 9 / 2 / 1 reservas, 28 noites, `7566.00`, média `270.21`, meses e top 5 iguais).
- Dataset real: 300 reservas (235 confirmadas, 31 canceladas, 34 no-show), 1347 noites, `396170.00` BRL, média `294.11`.
- Testes fornecidos (5) e adicionais (6): todos passando.
- `pytest -q`: PS C:\Users\victo\Projetos\onfly-desafio> python -m pytest -q
.....                                                                                                                                                                   [100%]
5 passed in 0.58s
