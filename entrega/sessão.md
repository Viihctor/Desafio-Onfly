onfly-desafio-20261114T154914Z-1-001.zip
ZIP

Estou iniciando o desafio técnico da Onfly.
Analise integralmente o README.md, a estrutura do projeto, os arquivos Python, os testes existentes e os arquivos de dados.
Quero que você produza:

Uma explicação do objetivo do software.
Um resumo de cada módulo e sua responsabilidade.
Um levantamento dos requisitos funcionais e regras de negócio.
Um levantamento dos requisitos técnicos e restrições.
Uma explicação do fluxo atual de processamento dos dados.
Uma relação dos testes fornecidos e o que cada um valida.
Uma lista inicial de possíveis áreas problemáticas, sem presumir que sejam defeitos confirmados.

Já li o README e todo o código-fonte. Agora leio os testes e os arquivos de dados, depois monto a análise completa.

Analisei o README, os 15 arquivos Python, os 5 testes e os 3 CSVs. Não alterei nada. Também executei o CLI como está na amostra e no dataset completo, para embasar o item 7. O pytest não está instalado neste ambiente e não consegui instalá-lo, então não rodei a suíte.

1. Objetivo do software

É um CLI em Python 3.11+, só com biblioteca padrão, que consolida a exportação de reservas de hotel de um cliente da Onfly. Ele lê o CSV de reservas e a tabela de câmbio e gera um relatório JSON com:

totais gerais e contagens por status;
consolidação por mês de check-in;
ranking de hotéis por valor faturado em BRL.

Segundo o README, hoje ele “entrega números errados”. A tarefa é encontrar e corrigir os defeitos, sem alterar os testes, o contrato do CLI, o schema do relatório nem os CSVs reservas_hoteis.csv e taxas_cambio.csv.

2. Módulos e responsabilidades
Módulo	Responsabilidade
__main__.py	Ponto de entrada de python -m onfly_report; chama cli.main().
cli.py	Argumentos (--input, --rates, --output, --top), validação de --top >= 1, orquestração, gravação do JSON, resumo no stdout e códigos de saída (0, 1, 2).
models.py	Dataclasses imutáveis RawRow (linha crua com número da linha) e Reservation (tipada).
exceptions.py	DomainError e InputFileError; esta última vira exit code 2.
logging_config.py	Logging em stderr, deixando o stdout para o resumo.
repositories/csv_reservation_repository.py	Lê o CSV de reservas, valida as colunas obrigatórias e devolve RawRows sem aplicar regra de negócio.
repositories/csv_rate_repository.py	Lê a tabela de câmbio e valida colunas, taxa positiva e arquivo não vazio. Devolve dict[moeda, Decimal].
services/row_parser.py	Converte RawRow em Reservation (datas, int, Decimal). Erro de formato vira InputFileError.
services/status_policy.py	is_billable(status): decide se a reserva entra nos totais.
services/stay_calculator.py	calculate_room_nights(checkin, checkout, rooms).
services/currency_converter.py	convert_to_brl(amount, currency, rates).
services/report_builder.py	Orquestra tudo: parse, contagem por status, filtro, agregação (total, mês, hotel), ordenação, formatação monetária e montagem do dict final.
3. Requisitos funcionais e regras de negócio

Status

Só CONFIRMED é faturável.
CANCELLED e NO_SHOW contam em total_reservations e nas contagens por status. Ficam fora de noites, valores, by_month e top_hotels.

Noites

room_nights = (checkout - checkin).days * rooms, diferença não inclusiva (10/01 a 13/01 = 3 noites).
Vale também na virada de mês e de ano.

Câmbio

valor_brl = amount * rate_to_brl.
Todas as moedas do dataset têm taxa.

Totais

total_reservations é o número de reservas no arquivo.
As três contagens por status somam esse total.
total_room_nights, total_amount_brl e average_nightly_rate_brl usam só CONFIRMED.
average_nightly_rate_brl = total_amount_brl / total_room_nights, ou "0.00" se não houver noites.

Agregações e ordenação

by_month usa o mês do check-in, em ordem ascendente.
top_hotels tem --top itens (ou menos, se houver menos hotéis). A ordem é total_brl decrescente, com desempate por hotel_id ascendente.

Formato

report_version é sempre "1.0".
Dinheiro é string com 2 casas e sem separador de milhar.
generated_at usa offset -03:00 e é ignorado na comparação.

Contrato do CLI

--output tem padrão output/report.json e o diretório é criado.
--top tem padrão 5 e deve ser um inteiro ≥ 1.
Exit code 0 é sucesso, 2 é entrada ou argumento inválido, 1 é erro inesperado.
4. Requisitos técnicos e restrições
Python 3.11+, sem dependências de runtime. pytest==8.3.4 e ruff==0.8.4 são só de desenvolvimento.
Dinheiro com Decimal, nunca float. Soma exata e arredondamento só na serialização (2 casas, half-up).
Ordem de processamento: ler → converter tipos → contar status → filtrar CONFIRMED → agregar.
Datas civis YYYY-MM-DD. Os dados são declarados consistentes, então o defeito está no código e não no CSV.
Ruff com linha de 120 colunas e regras E, F e I.
Não pode mudar: os testes fornecidos, o comando e os argumentos do CLI, o schema e a semântica do relatório, e os dois CSVs de dados.
Pode mudar: a organização interna do código, logs e testes novos (test_*.py).
Entrega: repositório público no GitHub com entrega/README-ENTREGA.md. Os defeitos corrigidos devem trazer causa raiz em uma frase cada, o método de investigação, como validar e o resultado do pytest. Se usar IA, vai também entrega/uso-de-ia/ com a sessão exportada e a declaracao.md.

Dados

reservas_hoteis.csv tem 300 reservas, 12 hotéis, uma só empresa (EMP-001) e check-ins de 2024-12 a 2025-06.
Status: 235 CONFIRMED, 31 CANCELLED e 34 NO_SHOW.
Moedas: 185 BRL, 70 USD e 45 EUR.
Quartos: 1, 2 ou 3 por reserva.
93 estadias cruzam a virada de mês.
Não há IDs duplicados, datas inválidas nem checkout ≤ checkin.
taxas_cambio.csv tem BRL=1, USD=5,42 e EUR=5,88.
exemplo-validacao.csv tem 12 reservas com resultado esperado documentado no README.
5. Fluxo atual de processamento
cli.main() configura o logging, faz o parse dos argumentos e valida --top.
ReportBuilder.build() chama CsvReservationRepository.read(), que devolve RawRows, e depois RowParser.parse() em cada linha, gerando Reservations.
CsvRateRepository.read() carrega o dict de taxas.
Todas as reservas são contadas por status.
As reservas passam por is_billable().
Para cada reserva faturável, o código calcula room_nights e amount_brl, soma nos totais, no dict mensal (chave YYYY-MM do check-in) e no dict por hotel (chave hotel_id).
Calcula a média, ordena os meses e ordena os hotéis, e corta no --top.
Monta o dict do relatório. O format_money quantiza em half-up só nesta etapa.
cli grava o JSON em UTF-8 com indentação de 2 espaços, imprime o resumo e devolve 0. InputFileError devolve 2, qualquer outra exceção devolve 1.
6. Testes fornecidos

Todos passam pelo tests/helpers.py, que monta CSVs temporários, roda o CLI via subprocess e lê o JSON. As taxas usadas são BRL=1, USD=5,42 e EUR=5,88.

Arquivo	O que valida
test_currency.py	100 BRL + 100 USD dá total_amount_brl == "642.00". Ou seja, multiplica pela taxa em vez de dividir.
test_nights.py	Check-in 30/01 e checkout 02/02 com 1 quarto dá total_room_nights == 3, cobrindo a virada de mês.
test_status_rules.py	Com 1 CONFIRMED, 1 CANCELLED e 1 NO_SHOW: total 3, uma reserva em cada contagem, valor 100.00 e 1 noite. Só CONFIRMED entra nos totais.
test_top_hotels.py	Com 4 hotéis e --top 3, devolve H01, H02 e H03, com valores 1000, 900 e 800 em ordem decrescente.
test_regression_report.py	Ponta a ponta com 3 reservas: 3 total, 2 confirmadas, 1 cancelada, 5 noites, 1542.00 e média 308.40.
7. Possíveis áreas problemáticas (hipóteses)

Rodei o CLI como está. Na amostra de validação ele deu -33 noites e 5962.41 de total, contra 28 noites e 7566.00 esperados. No dataset completo contou 200 reservas em vez de 300. Isso indica que há problemas reais, mas cada ponto abaixo ainda é uma hipótese a confirmar na correção.

status_policy.is_billable usa status != "CANCELLED". Isso deixa NO_SHOW passar como faturável. Na saída da amostra, o hotel H05 (só um NO_SHOW de 300) aparece no ranking.
stay_calculator calcula checkout.day - checkin.day. Isso compara o dia do mês, não as datas, e gera valores negativos quando a estadia cruza o mês. É o que produz o -33.
currency_converter divide em vez de multiplicar. Além disso, passa por float e depois por Decimal(str(...)), o que contradiz a regra de exatidão. É o que reduz o total da amostra.
csv_reservation_repository.read tem um break em 200 linhas. Isso trunca silenciosamente o arquivo e afeta todos os totais e contagens.
report_builder, ordenação de top_hotels:
A chave é (amount, hotel_id) em ordem ascendente, e o spec pede valor decrescente.
O desempate por hotel_id precisa continuar ascendente, então não basta inverter a lista inteira.
O ranking deve usar o valor exato e não o arredondado.
average_nightly_rate_brl: com noites negativas, hoje cai no ramo "0.00". É consequência do item 2, mas vale revalidar depois da correção.
Tratamento de status e moeda: o parser faz strip() mas não normaliza caixa.
O status_counts aceita status desconhecidos sem exibi-los, então os três totais podem não fechar com total_reservations.
rates[currency] levanta KeyError para moeda ausente ou em caixa diferente, o que viraria exit code 1 em vez de 2.
O dataset fornecido não exercita nenhum desses casos.
Arredondamento half-up: format_money parece correto. Mas o item 3 pode mascarar casos de meio centavo, que só aparecem depois de eliminar o float.
README e CLI:
O README exportado tem sublinhados escapados (onfly\_report, \\ nos comandos). Copiar e colar os comandos falha.
O README diz que no início “todos falham”. Convém conferir isso com o pytest depois de instalá-lo.
Lacunas de cobertura nos testes fornecidos:
virada de ano (dez → jan);
empate no ranking e valor padrão de --top;
arredondamento half-up;
rooms > 1;
arquivos com mais de 200 linhas;
by_month;
códigos de saída.
Faz sentido cobrir esses casos com testes novos em arquivos separados, já que os fornecidos não podem ser alterados.

Posso seguir para a correção dos pontos 1 a 5, com testes adicionais para as lacunas do item 10 e o rascunho do entrega/README-ENTREGA.md.

faça um readme para eu colocar no github

O README.md está pronto para o repositório. Ele descreve o projeto como o desafio o define: objetivo, uso do CLI, formato do JSON, regras de negócio, estrutura, testes e a amostra de validação.

Correções ainda não aplicadas: o README não afirma que os defeitos foram corrigidos nem que o pytest passa. Quando você fizer as correções, vale rodar o CLI na amostra e comparar com a tabela de resultado esperado.
Sobre a entrega: o README aponta para entrega/README-ENTREGA.md, onde o desafio pede os defeitos, a causa raiz, a investigação e o resultado do pytest.
Link do clone: troque <seu-usuario> na seção de instalação pelo seu usuário do GitHub.
Comandos: estão sem os sublinhados escapados que o README original tinha, então podem ser copiados e colados.

Se quiser, preparo agora o entrega/README-ENTREGA.md ou sigo para a correção dos defeitos.

README
Documento
·MD

Para cada possível defeito:

Identifique o arquivo e a função envolvidos.
Explique o comportamento atual.
Explique o comportamento esperado.
Identifique a provável causa raiz.
Mostre qual regra de negócio está sendo violada.
Explique como reproduzir o problema.
Indique qual teste existente ou adicional poderia detectar o defeito.

Vou reproduzir cada defeito antes de descrevê-lo, para que os comportamentos atuais que vou citar sejam medidos e não só deduzidos do código.

Reproduzi cinco defeitos chamando as funções e o CLI, sem alterar o projeto. Um sexto ponto, de robustez, vem só da leitura do código e não foi reproduzido. Os defeitos se somam: o CLI na amostra de validação dá -33 noites e 5962.41, e no dataset completo dá 200 reservas, 153 confirmadas e -1393 noites.

D1 — NO_SHOW tratado como faturável
Local: services/status_policy.py, função is_billable.
Comportamento atual: retorna status != "CANCELLED". Medido: CONFIRMED→True, CANCELLED→False, NO_SHOW→True. Reservas NO_SHOW entram em noites, valores, by_month e top_hotels. Na amostra, o H05 (só um NO_SHOW de 300) aparece no ranking.
Esperado: só CONFIRMED é faturável.
Causa raiz provável: a política foi escrita como lista de exclusão com um único status. Qualquer outro status passa, inclusive NO_SHOW e valores desconhecidos.
Regra violada: “Só CONFIRMED é faturável. CANCELLED e NO_SHOW […] não entram em noites, valores, by_month nem top_hotels.”
Reprodução:
bash
PYTHONPATH=src python -c "from onfly_report.services.status_policy import is_billable as b; print(b('NO_SHOW'))"

Deveria imprimir False, mas imprime True.
7. Testes:

test_status_rules.py detecta o defeito: o NO_SHOW de 20.00 elevaria o total para 120.00 e as noites para 2.
Adicional sugerido: um teste unitário de is_billable com os três status e um desconhecido, e um teste do ranking com um hotel que só tenha NO_SHOW.
D2 — Cálculo de noites usa o dia do mês
Local: services/stay_calculator.py, função calculate_room_nights.
Comportamento atual: calcula (checkout.day - checkin.day) * rooms. Medido: 30/01→02/02 dá -28 (esperado 3) e 30/12→02/01 dá -28. Dentro do mesmo mês o resultado sai certo (10/01→13/01 com 2 quartos dá 6), o que esconde o defeito. Efeito: noites negativas em totais e by_month. Com total_room_nights negativo, a média cai no ramo "0.00".
Esperado: (checkout - checkin).days * rooms, inclusive na virada de mês e de ano.
Causa raiz provável: o código subtrai o atributo .day (inteiro 1–31) em vez de subtrair as datas.
Regra violada: “Noites: room_nights = (checkout_date - checkin_date).days * rooms. […] Vale também na virada de mês/ano.”
Reprodução:
bash
PYTHONPATH=src python -c "
from datetime import date
from onfly_report.services.stay_calculator import calculate_room_nights as n
print(n(date(2025,1,30), date(2025,2,2), 1))"

Deveria imprimir 3, mas imprime -28. No dataset, 93 das 300 estadias cruzam o mês.
7. Testes:

test_nights.py detecta o defeito.
O teste de regressão não detecta, porque todas as suas estadias ficam dentro de janeiro.
Adicional sugerido: virada de ano (30/12→02/01), rooms > 1 com virada de mês, e ano bissexto (28/02→01/03 em 2024).
D3 — Câmbio dividido pela taxa e passando por float
Local: services/currency_converter.py, função convert_to_brl.
Comportamento atual: faz Decimal(str(float(amount) / float(rate))). Medido: 100 USD → 18.4501… (esperado 542.00) e 150 EUR → 25.5102… (esperado 882.00). Há dois problemas:
Operação invertida: divide em vez de multiplicar.
Precisão: converte para float e volta para Decimal.
Esperado: amount * rate_to_brl, tudo em Decimal, sem arredondar nas parcelas.
Causa raiz provável: dois erros independentes na mesma linha: operador errado e conversão desnecessária para float.
Regras violadas: “Converta com amount * rate_to_brl” e “Use Decimal, nunca float, no dinheiro.”
Reprodução:
bash
PYTHONPATH=src python -c "
from decimal import Decimal as D
from onfly_report.services.currency_converter import convert_to_brl as c
print(c(D('100.00'),'USD',{'USD':D('5.420000')}))"

Deveria imprimir 542.00…, mas imprime 18.45018450184502.

Nuance sobre o float: num cálculo à parte, mantendo a multiplicação, o caminho float difere do exato em 19 das 235 linhas confirmadas, e em uma delas após arredondar a linha. O total final arredondado coincide nos dois caminhos (396170.00). Então o defeito do float é uma violação da regra e um risco real, mas não é o que quebra os números do dataset. A divisão é que quebra.
Testes:
test_currency.py detecta o defeito.
test_regression_report.py também falha por aqui: o USD vira 18.45 em vez de 542.00.
Adicional sugerido: um teste de meio centavo exato (o dataset tem 3 casos, como RSV-0111, RSV-0154 e RSV-0299) para fixar o half-up, e um teste de que a conversão devolve Decimal, sem float no caminho.
D4 — Leitura do CSV truncada em 200 linhas
Local: repositories/csv_reservation_repository.py, método CsvReservationRepository.read.
Comportamento atual: o laço faz if len(rows) == 200: break. Medido: o repositório devolve 200 linhas de um arquivo com 300, sem erro nem aviso. O truncamento afeta total_reservations, as contagens por status e todos os agregados.
Esperado: ler todas as reservas do arquivo.
Causa raiz provável: um limite fixo que nada no contrato prevê, provavelmente deixado de depuração ou de amostragem.
Regra violada: “total_reservations = número de reservas no arquivo.”
Reprodução:
bash
PYTHONPATH=src python -m onfly_report --input data/reservas_hoteis.csv \
  --rates data/taxas_cambio.csv --output /tmp/r.json

O resumo mostra Reservas: 200, mas o arquivo tem 300 linhas de dados.
7. Testes:

Nenhum teste fornecido detecta, porque todos usam no máximo 4 linhas.
Adicional sugerido: gerar 250 reservas e verificar total_reservations == 250. Também vale um teste de regressão com o reservas_hoteis.csv real, comparando as contagens: 300 reservas, 235 confirmadas, 31 canceladas e 34 no-show.
D5 — Ranking de hotéis em ordem crescente
Local: services/report_builder.py, método ReportBuilder.build, na variável ranked_hotels.
Comportamento atual: ordena por (amount, hotel_id) ascendente. Medido, com --top 3 e hotéis de 1000, 900, 800 e 700: devolve H04 (700), H03 (800), H02 (900). O ranking mostra os piores hotéis.
Esperado: total_brl decrescente, com desempate por hotel_id ascendente.
Causa raiz provável: falta a inversão do critério de valor. Um reverse=True simples não resolve, porque inverteria também o desempate. A chave correta é algo como (-amount, hotel_id).
Regra violada: “top_hotels: total_brl decrescente; empate por hotel_id ascendente.”
Reprodução: o CLI sobre a amostra de validação, com --top 5, mostra H09 (25.51) antes de H05 (300.00), em ordem crescente. O esperado é H01, H02, H12, H10, H09.
Testes:
test_top_hotels.py detecta o defeito.
Ele não cobre o desempate. Adicional sugerido: dois hotéis com o mesmo total e IDs H02 e H01 (esperado H01 primeiro), e um teste de que o ranking compara o valor exato e não o arredondado.
D6 — Robustez de status e moeda (hipótese, só por leitura do código)
Local: services/row_parser.py (RowParser.parse) e services/report_builder.py (build, status_counts).
Comportamento atual (inferido):
O parser faz strip() mas não normaliza a caixa de status e currency.
Um status fora dos três conhecidos entra em status_counts, mas não aparece no relatório. A soma das três contagens deixa de fechar com total_reservations.
rates[currency] levanta KeyError para uma moeda ausente ou em caixa diferente (por exemplo usd), o que viraria exit code 1 e não 2.
Esperado: não está definido no README. O texto afirma que os dados são consistentes e que todas as moedas têm taxa.
Causa raiz provável: o código assume dados limpos, e o repositório de câmbio normaliza para maiúsculas enquanto o parser de reservas não.
Regra violada: nenhuma com certeza. É um risco de robustez e não um defeito do dataset fornecido. Corrigir é opcional, mas é barato e pode ser um diferencial.
Reprodução: não reproduzi. Exigiria um CSV de teste com currency=usd ou status=OTHER.
Testes: nenhum existente. Adicional sugerido: moeda sem taxa deve dar exit code 2, status desconhecido deve ser tratado de forma explícita, e o CLI com colunas ausentes deve dar exit code 2.
Pontos que não são defeitos independentes
Ponto	Situação
average_nightly_rate_brl	Hoje sai "0.00" por causa das noites negativas (D2). A fórmula em si parece correta. Revalidar depois de corrigir D2, e testar o caso de zero noites.
format_money (half-up)	Parece correto. Só se confirma depois de remover o float (D3).
README com \_ escapado	Problema de documentação, não de código. O README que preparei para o GitHub já está sem os escapes.
Quais testes pegam cada defeito
Defeito	Testes fornecidos que falham	Falha por causa de
D1	test_status_rules	só D1
D2	test_nights	só D2
D3	test_currency, test_regression_report	só D3
D4	nenhum	—
D5	test_top_hotels	só D5

Cada teste fornecido falha por um único defeito, o que facilita corrigir um de cada vez. A exceção é o D4, que só um teste adicional pega.

Depois das correções, o total esperado para as 235 reservas confirmadas do dataset completo é 396170.00. Esse valor vem de um cálculo meu feito à parte, só com multiplicação em Decimal, e serve como conferência cruzada. O resultado final do CLI ainda precisa ser conferido com o número de noites.

Posso aplicar as correções de D1 a D5 e escrever os testes adicionais, deixando os fornecidos intactos.

agora ao fim faça a declaração para eu anexar ao projeto:

# Declaração de uso de IA

- **Ferramenta e modelo:** C
- **O que a IA fez:**
- **O que eu fiz:**
- **Sessão:** exportação da conversa em `sessao.md` neste mesmo diretório.

A declaracao.md está pronta, seguindo o seu modelo. Deixei quatro trechos marcados com [PREENCHER] na seção “O que eu fiz”, porque só você sabe o que revisou, validou e decidiu. Não quis inventar isso em seu nome, já que a declaração é um documento de avaliação.

O que a IA fez: descrevi só o que aconteceu nesta conversa, incluindo as limitações. Não rodei o pytest, que não estava instalado no meu ambiente, e ainda não apliquei correção alguma no código.
Modelo: informei Claude Sonnet 5.5, o modelo desta conversa. Como o modelo pode ser trocado no meio de uma sessão, confira se você não mudou de modelo e ajuste se for o caso.
Atualização: se você aplicar as correções comigo, ou sozinho, depois deste ponto, atualize os dois lados da declaração. Posso refazê-la quando terminarmos.
Sessão: o arquivo sessao.md precisa ser gerado por você. Copie a conversa inteira e salve em entrega/uso-de-ia/, junto com a declaracao.md.
Declaracao
Documento
·MD