# Lattice Agreement

Псевдокод алгоритма из лекции (его обработал ИИ из дока, поэтому возможно где-то косячный):
```plaintext
Local variables:
1:  bufferedValue = u0       // join of known proposals
2:  proposedValue = u0       // the current proposal
3:  acceptedValue = u0       // join of accepted proposals
4:  readyToLearn = false     // ready to adopt a learn value

Upon propose(v)              // process p proposes v:
5:  t++                      // sequence number of the proposal
6:  readyToLearn := false
7:  proposedValue := v
8:  bufferedValue := bufferedValue ⊔ v
9:  send [PROPOSE, v, t, p] to all

Upon received [PROPOSE, v′, t′, p′]:
10: if acceptedValue ⊑ v′ then
11:     acceptedValue := v′
12:     send [ACK, v′, t′, p′] to p′           // accept the proposal
13: else
14:     acceptedValue := acceptedValue ⊔ v′
15:     send [NACK, acceptedValue, t′, p′] to p′  // reject the proposal

Upon received [ACK/NACK, v, t, p] from a quorum:
16: if no [NACK, v, t, p] received then
17:     send [LEARN, bufferedValue] to all
18:     return bufferedValue                  // learn a new value
19: else
20:     t++
21:     if not readyToLearn then
22:         readyToLearn := true
23:         proposedValue := bufferedValue
24:         send [PROPOSE, bufferedValue, t, p] to all  // send a new proposal

Upon received [NACK, v, t, p]:
25: bufferedValue := bufferedValue ⊔ v

Upon received [LEARN, v′]:
26: if proposedValue ⊑ v′ and readyToLearn then
27:     send [LEARN, v′] to all                // "reliable" broadcast
28:     return v′                              // adopt a new value
```

## 1. Weak Counter через Lattice Agreement

**Решётка**: 

$\mathcal{L} = \mathbb{N}$,
 $\sqsubseteq \;=\; \le$, $\sqcup \;=\; \max$.
Натуральные числа, используем сравнение меньше-равно, для объединения используем ф-ию max()

**increment()**: процесс читает своё последнее известное значение $v$ и вызывает `propose(v + 1)`. По свойству Validity результат гарантированно $\ge v+1$.

**get()**: процесс вызывает `propose(v)` с текущим известным значением $v$ (no-op-предложение) и дожидается, пока оно будет выучено; возвращает итоговое значение из `LearntValue()`. Это гарантирует синхронизацию со всеми инкрементами, произошедшими до начала чтения, и линеаризуемость.

Поскольку $\sqcup = \max$, все выученные значения образуют монотонно возрастающую цепочку чисел – то есть ведут себя как счётчик. Но конкурентные инкременты теоретически могут «перекрыть» друг друга.

---
## 2. Replicated Register (в духе ABD) через Lattice Agreement
TBD

---
## 3. Asset Transfer через Lattice Agreement
TBD
---

## 4. За счёт чего выполняется Comparability
По псевдокоду алгоритма:

1. Действие `Accept` срабатывает только при `acceptedValue ⊑ proposedValue`, поэтому **у одного acceptor'а** все когда-либо принятые значения образуют монотонно возрастающую **цепочку**.
2. Значение становится «выбранным» (chosen), только если его приняло **большинство** acceptor'ов: $\ge \lceil (N_a+1)/2 \rceil$.
3. Любые два кворума **обязательно пересекаются** хотя бы в одном acceptor'е (по принципу Дирихле, так как каждый кворум — больше половины).
4. Получается, что для любых двух выбранных значений $u$, $v$ найдётся общий acceptor, принявший оба — а у него (см. п.1) все принятые значения сравнимы по построению. Следовательно, $u \sqsubseteq v$ или $v \sqsubseteq u$.

Появление двух несравнимых выбранных значений невозможно, потому что мы обязаны делить общего «свидетеля»-acceptor'а, чья история принятых значений всегда является цепочкой.

---
## 5. Доказательство: хотя бы один процесс завершит propose
TBD
---
## 6. Что сломается при удалении сообщений LEARN
TBD
---
## 7. Адаптация к Byzantine отказам, нужен ли PKI
TBD
