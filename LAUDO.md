# LAUDO TÉCNICO — Operação Resgate: Detetives de Bugs

**Projeto:** Central de Achados e Perdidos (Flutter + Firebase Auth + Firestore)
**Dupla/Aluno:** João Pedro Cardoso Duarte

## Diagnóstico inicial (Missão 1)

Caminho investigado: `main` → `GatePage` → `CentralPage` → `DetalhePage`.
Antes dos 12 bugs de lógica, quatro **problemas de ambiente** impediam o app de compilar/analisar:

| Ambiente | Arquivo | Problema | Correção |
|---|---|---|---|
| A | `lib/firebase_options.dart` | Arquivo ausente (é gerado pelo FlutterFire CLI e não vem no ZIP), então `main.dart` não compilava | Rodei `flutterfire configure` apontando para o meu projeto Firebase |
| B | `pubspec.yaml` | `analysis_options.yaml` inclui `flutter_lints`, mas não havia `dev_dependencies` | Adicionei `flutter_test` e `flutter_lints` em `dev_dependencies` |
| C | `test/widget_test.dart` | Teste padrão do "contador" importava `package:bugsresolver` (nome errado) e `MyApp` (inexistente) | Substituí por testes do modelo `Ocorrencia`, usando `package:central_achados` |
| D | `lib/pages/central_page.dart` (subtitle) | String com quebra de linha literal dentro de aspas simples: erro de compilação | Troquei a quebra literal pelo escape `\n` |

---

## Bug 1
- **Arquivo:** `lib/pages/central_page.dart` (~linha 86)
- **Sintoma:** o app não compilava; ao tocar numa ocorrência não abria o detalhe.
- **Causa:** `DetalhePage()` era chamada sem o parâmetro obrigatório `ocorrencia`.
- **Correção:** `DetalhePage(ocorrencia: o)`.
- **Teste:** `flutter analyze` sem o erro de argumento; ao tocar em um card, a tela "Investigação" abre com o título/local corretos.

## Bug 2
- **Arquivo:** `lib/models/ocorrencia.dart`
- **Sintoma:** o botão "Marcar como resolvido" não aparecia para o dono, e a checagem de autorização falhava sempre.
- **Causa:** `autorId` era lido do campo `'autorNome'` em vez de `'autorId'`.
- **Correção:** `autorId: d['autorId'] as String? ?? ''`.
- **Teste:** teste unitário `fromMap lê autorId...` (`flutter test`); na conta dona o botão aparece.

## Bug 3
- **Arquivo:** `lib/models/ocorrencia.dart`
- **Sintoma:** casos resolvidos apareciam como "Em aberto" e vice-versa.
- **Causa:** `resolvido: !(d['resolvido'] ?? false)` — o `!` invertia o valor.
- **Correção:** `resolvido: d['resolvido'] as bool? ?? false`.
- **Teste:** testes unitários (`resolvido false → false`, `true → true`); no app, novo registro aparece "Em aberto".

## Bug 4
- **Arquivo:** `lib/services/central_service.dart` (`registrar`)
- **Sintoma:** o dono não conseguia alterar o status do próprio caso.
- **Causa:** `autorId` era gravado com `u.email`, mas `alterarStatus` compara com `u.uid`.
- **Correção:** `'autorId': u.uid`.
- **Teste:** registrei uma ocorrência e conferi no Firestore que `autorId` = UID do usuário; o botão de status funcionou.

## Bug 5
- **Arquivo:** `lib/pages/central_page.dart` (`cadastrar`)
- **Sintoma:** era possível registrar com título vazio ou com local vazio.
- **Causa:** validação usava `&&` (só bloqueava se **ambos** estivessem vazios).
- **Correção:** trocado por `||`.
- **Teste:** tentei registrar só com título e só com local: aparece "Informe título e local." e nada é gravado.

## Bug 6
- **Arquivo:** `lib/pages/central_page.dart` (`SwitchListTile`)
- **Sintoma:** o switch "Somente não resolvidos" funcionava ao contrário (ligado = desligado).
- **Causa:** `onChanged: (value) => somenteAbertos = !value`.
- **Correção:** `somenteAbertos = value`.
- **Teste:** ao ligar o switch, ele fica ligado e o filtro é aplicado.

## Bug 7
- **Arquivo:** `lib/pages/central_page.dart` (build do `StreamBuilder`)
- **Sintoma:** o filtro mostrava apenas os casos **resolvidos**.
- **Causa:** `todos.where((o) => o.resolvido)` — condição invertida.
- **Correção:** `todos.where((o) => !o.resolvido)`.
- **Teste:** com um caso resolvido e um aberto, o filtro ligado mostra só o aberto.

## Bug 8
- **Arquivo:** `lib/services/central_service.dart` (`acompanhar`)
- **Sintoma:** os registros mais antigos apareciam primeiro; o novo ficava no fim da lista.
- **Causa:** `orderBy('criadoEm', descending: false)`.
- **Correção:** `descending: true`.
- **Teste:** registrei dois objetos em sequência; o mais recente aparece no topo.
- **Obs.:** o `orderBy('criadoEm')` pode pedir criação de índice no console do Firebase só se houver filtros adicionais (não é o caso aqui).

## Bug 9
- **Arquivo:** `lib/services/central_service.dart` (`alterarStatus`)
- **Sintoma:** clicar em "Marcar como resolvido" não mudava nada.
- **Causa:** gravava `item.resolvido` (o valor atual) em vez do valor invertido.
- **Correção:** `update({'resolvido': !item.resolvido})`.
- **Teste:** marquei um caso como resolvido: o card passou a "Resolvido" nas duas contas; reabri e voltou a "Em aberto".

## Bug 10
- **Arquivo:** `lib/services/central_service.dart` (`enviarPista`)
- **Sintoma:** a pista enviada não aparecia no caso; todas as pistas iam para um único lugar.
- **Causa:** gravava em `ocorrencias/pistas/pistas` (documento literal `'pistas'`) em vez do ID da ocorrência. O stream lê de `ocorrencias/{id}/pistas`.
- **Correção:** `.doc(ocorrenciaId).collection('pistas')`.
- **Teste:** na conta B enviei uma pista no caso X; no Firestore ela aparece em `ocorrencias/X/pistas`, e as duas contas veem a pista só no caso X.

## Bug 11
- **Arquivo:** `lib/pages/detalhe_page.dart` (`enviar`)
- **Sintoma:** ao digitar uma pista e clicar em enviar, nada acontecia.
- **Causa:** `if (pista.text.trim().isNotEmpty) return;` — saía justamente quando havia texto.
- **Correção:** `if (mensagem.isEmpty) return;`.
- **Teste:** com texto, a pista é enviada; com campo vazio, nada é enviado.

## Bug 12
- **Arquivo:** `lib/pages/detalhe_page.dart` (`enviar`)
- **Sintoma:** mesmo quando enviava, a pista chegava com mensagem vazia.
- **Causa:** `enviarPista(widget.ocorrencia.id, '')` — texto fixo vazio.
- **Correção:** passa `mensagem` (texto digitado, com `trim`).
- **Teste:** a pista aparece na lista com o texto digitado e o nome do autor.

---

## Segurança (Missão 3)

Não removi autenticação nem desativei regras. Adicionei o arquivo `firestore.rules` com regras estritas:
- só usuários logados leem/escrevem;
- só se cria ocorrência com `autorId == request.auth.uid`;
- só o **dono** altera uma ocorrência, e **apenas o campo `resolvido`**;
- ninguém apaga ocorrências; pistas só podem ser criadas (com o próprio uid) e não editadas/apagadas.

**Teste de autorização:** na conta B, o botão de status não aparece em casos da conta A (e, mesmo forçando a chamada, o Firestore recusa por regra).

## Evidência

Captura de tela das duas contas vendo a mesma ocorrência: `prints/duas_contas.png`.
