# Conta Bancária / Transferências Concorrentes (.NET 10)

Projeto console em C# (`.NET 10`) desenvolvido para a disciplina de Concorrência de Software, modelando o problema clássico de **Conta Bancária com Transferências Concorrentes**.

---

## 1. Problema Escolhido

**Conta Bancária / Transferências Concorrentes:** múltiplas operações financeiras (depósitos, saques e, principalmente, transferências entre duas contas) ocorrem simultaneamente sobre um conjunto compartilhado de contas bancárias.

---

## 2. Threads e Recursos Disputados

### Quem são as threads?
- **Threads de Transferência (Clientes / Operadores Bancários):** `N` threads concorrentes que executam repetidamente transferências de valores entre pares de contas escolhidos aleatoriamente (conta de origem → conta de destino).

### Quais recursos elas disputam?
- **Contas bancárias compartilhadas** (especificamente o campo `Saldo` de cada conta envolvida em uma transferência).
- Em cada transferência, uma thread precisa acessar **dois recursos compartilhados simultaneamente**: a conta de **origem** (leitura de saldo + débito) e a conta de **destino** (crédito).

---

## 3. Regras de um Acesso Seguro

Para que o sistema seja considerado correto e seguro sob concorrência, as seguintes regras devem ser respeitadas:

1. **Exclusão mútua por conta:** duas threads não podem ler e modificar o saldo da mesma conta simultaneamente.
2. **Atomicidade da transferência:** a verificação de saldo, o débito na conta de origem e o crédito na conta de destino devem ocorrer de forma indivisível do ponto de vista das demais threads.
3. **Conservação de valor (Invariante global):** a soma total dos saldos de todas as contas no banco deve permanecer constante antes, durante (em estados consistentes) e após todas as transferências.
4. **Integridade de saldo:** nenhuma transferência pode ser efetivada se a conta de origem não possuir saldo suficiente (nenhuma conta pode ficar com saldo negativo).
5. **Ausência de deadlock (Progresso garantido):** a aquisição simultânea de múltiplas contas não pode gerar espera circular entre threads.

---

## 4. Soluções Ingênuas Planejadas

Serão implementadas **2 soluções ingênuas** para demonstrar falhas clássicas de concorrência:

1. **Solução Ingênua 1 — Sem sincronização (acesso concorrente livre):**
   - As threads leem o saldo, validam se há fundos, debitam da origem e creditam no destino sem nenhum mecanismo de trava.
   - **Problema provocado:** **Condição de Corrida (*Race Condition*) / Atualização Perdida (*Lost Update*)**. Duas threads podem ler o mesmo saldo simultaneamente e sobrescrever o resultado uma da outra, causando **divergência no saldo total do banco** (dinheiro criado ou perdido) e **saldos negativos**.

2. **Solução Ingênua 2 — Trava (`lock`) na ordem `origem` → `destino`:**
   - Cada conta possui sua própria trava, e a thread adquire primeiro o `lock` da conta de origem e depois o `lock` da conta de destino.
   - **Problema provocado:** **Deadlock (Impasse)**. Quando a Thread 1 transfere de `A → B` (trava `A` e aguarda `B`) ao mesmo tempo em que a Thread 2 transfere de `B → A` (trava `B` e aguarda `A`), ambas ficam bloqueadas indefinidamente em espera circular.

---

## 5. Solução Ideal e Verificações Automáticas

### Ferramenta da solução ideal
- **`System.Threading.Lock` (C# / .NET 10) com ordenação global de travas por `Id` da conta:**
  - Cada conta possui um objeto `Lock` próprio e um identificador único (`Id`).
  - Antes de efetuar a transferência, a thread ordena as duas contas pelo `Id` e adquire sempre primeiro a trava da conta de **menor `Id`** e depois a de **maior `Id`**.
  - Isso garante **exclusão mútua** com alta concorrência (transferências entre pares disjuntos de contas executam em paralelo) e **elimina completamente a possibilidade de deadlock**, quebrando a condição de espera circular.

### As 3 verificações automáticas do relatório
Ao final de cada execução, o programa executará automaticamente um relatório com 3 verificações:

1. **Verificação de Conservação do Saldo Total:** compara a soma inicial dos saldos de todas as contas com a soma final (`somaFinal == somaInicial`), detectando perda ou criação indevida de dinheiro.
2. **Verificação de Integridade de Saldo (Sem Saldo Negativo):** percorre todas as contas ao final da execução verificando se `conta.Saldo >= 0` para todas elas.
3. **Verificação de Progresso / Ausência de Deadlock:** monitora a conclusão de todas as threads dentro de um tempo limite (*timeout*) e confere se o número total de operações processadas (efetivadas + recusadas por saldo insuficiente) corresponde exatamente ao total esperado.

---

## 6. Como Executar

Requer o [.NET 10 SDK](https://dotnet.microsoft.com/download).

```bash
dotnet run
```
