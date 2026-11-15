# Declaração de uso de IA

- **Ferramenta e modelo:** Claude (Anthropic), usado pela interface de chat (claude.ai), modelo Claude Sonnet 5.5.

- **O que a IA fez:**
  - Leu o `README.md`, a estrutura do projeto, os módulos Python, os testes e os arquivos de dados (`reservas_hoteis.csv`, `taxas_cambio.csv`, `exemplo-validacao.csv`).
  - Explicou o objetivo do software, a responsabilidade de cada módulo, os requisitos funcionais e técnicos, as regras de negócio e o fluxo de processamento, e descreveu o que cada teste fornecido valida.
  - Executou o CLI no estado original, em uma cópia do projeto fora do repositório, para observar o comportamento atual. Também chamou as funções suspeitas isoladamente para reproduzir os problemas.
  - Levantou e analisou cinco defeitos, cada um com arquivo e função, comportamento atual e esperado, causa raiz, regra violada, reprodução e teste que o detectaria:
    - `NO_SHOW` tratado como faturável (`status_policy.py`);
    - cálculo de noites pelo dia do mês (`stay_calculator.py`);
    - câmbio dividido pela taxa e convertido por `float` (`currency_converter.py`);
    - leitura do CSV limitada a 200 linhas (`csv_reservation_repository.py`);
    - ranking de hotéis em ordem crescente (`report_builder.py`).
  - Apontou também uma hipótese de robustez (status e moeda sem normalização), marcada como não reproduzida.
  - Redigiu o `README.md` do repositório e este rascunho de declaração.
  - Limitações: não rodou o `pytest` (não estava instalado no ambiente da IA) e, até este ponto da sessão, não aplicou correções no código.

- **O que eu fiz:**
  - Forneci o projeto do desafio, defini o que queria analisar e pedi cada etapa à IA.
  - Revisei, conferi a análise dos defeitos contra o código, rodei o CLI e o `pytest` na minha máquina.
  - 'PS C:\Users\victo\Projetos\onfly-desafio> python -m pytest -q
.....                                                                                                                                                                   [100%]
5 passed in 0.58s'
  - Criei o repositório público no GitHub

- **Sessão:** [sessão transcrevida em sessão.md](https://claude.ai/share/31d87001-b63f-4171-bfbe-ad1329bd7892)
