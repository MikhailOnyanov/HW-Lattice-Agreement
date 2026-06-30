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

**Пример: P1, P2, P3
у всех в анчале:
acceptedValue = 0, t = 0**

Вариант 1, просто инкрементим

1. P1: вызывает propose(1): proposedValue:=1, bufferedValue:=0⊔1=1 -> шлёт PROPOSE(1) всем.

2. P1, P2, P3: получают PROPOSE(1), у каждого acceptedValue=0. Проверка acceptedValue ⊑ v′, то есть 0 ≤ 1 — верно -> Accept: acceptedValue:=1 -> шлют ACK к P1.

3. P1: получает ACK от кворума без единого NACK -> шлёт LEARN(1) всем -> P1 возвращает 1.

4. P2 и P3: получают LEARN(1): proposedValue=0⊑1 верно -> адаптируют 1, рассылают LEARN -> P2, P3 тоже возвращают 1.

Вариант 2, P2 инкрементит не зная про P1

5. P2 вызывает propose(2): proposedValue:=2, bufferedValue:=0⊔2=2 -> шлёт PROPOSE(2) всем.

6. Получатели проверяют acceptedValue ⊑ 2, то есть acceptedValue ≤ 2 — если acceptedValue уже 1 (после шага 2) или ещё 0, в обоих случаях 1 ≤ 2 и 0 ≤ 2 верны -> Accept -> ACK P2.

Вариант 3 с NACK, когда допустим кто-то долго спал и предлагает устаревшее предложение >
 
7. У процесса P уже acceptedValue = 2 (принял PROPOSE(2) раньше). Приходит устаревшее PROPOSE(0).
   Проверка acceptedValue ⊑ v′: 2 ≤ 0 — неверно:
   acceptedValue := acceptedValue ⊔ v′ <=> max(2, 0) = 2 -> шлёт NACK(2) отправителю.

8. Отправитель получает NACK(2): bufferedValue := bufferedValue ⊔ 2 = max(bufferedValue, 2).

9. Когда среди ответов кворума есть хотя бы один NACK: t:=t+1, proposedValue := bufferedValue
   -> отправитель шлёт новое PROPOSE(bufferedValue) всем заново.

10. Если новое предложение принимают все без NACK -> отправитель шлёт LEARN(bufferedValue) всем
    -> процесс возвращает bufferedValue.

---
## 2. Replicated Register (в духе ABD) через Lattice Agreement
TBD

---
## 3. Asset Transfer через Lattice Agreement
TBD
---

## 4. За счёт чего выполняется Comparability
По псевдокоду алгоритма:

1. Действие `Accept` срабатывает только когда у нас `acceptedValue ⊑ proposedValue`, поэтому у одного acceptor`а все когда-либо принятые значения образуют монотонно возрастающую цепочку.
2. Значение становится выбранным, только если его приняло **большинство** acceptor'ов: $\ge \lceil (N_a+1)/2 \rceil$.
3. Любые два кворума **обязательно пересекаются** хотя бы в одном acceptor'е (по принципу Дирихле, так как каждый кворум — больше половины).
4. Получается, что для любых двух выбранных значений $u$, $v$ найдётся общий acceptor, принявший оба — а у него (смотрим п.1) все принятые значения сравнимы по построению. Следовательно, $u \sqsubseteq v$ или $v \sqsubseteq u$.

Появление двух несравнимых выбранных значений невозможно, потому что мы обязаны делить общего «свидетеля»-acceptor'а, чья история принятых значений всегда является цепочкой.

---
## 5. Доказательство: хотя бы один процесс завершит propose

Идём от противного — пусть процесс p никогда не завершает propose(v).

1. Когда `p` рассылает [PROPOSE, v, ...] всем, то каждый получатель получит `v` в своё acceptedValue с: 
    1. Просто ринимаем напрямую;
    2. Объединяем через join перед NACK. 
    
    После этого v ⊑ acceptedValue у acceptor'а.

2. Значит, любое значение, выученное кем-то после получения предложения от `p`, уже содержит `v` (так как мы всё собираем через join из acceptedValue).

3. Получаем два случая:
   1. Кто-то ещё (q) бесконечно много раз успешно завершает propose -> рано или поздно выучивает значение, содержащее `v` -> рассылает LEARN -> `p` получает его и адаптирует -> p завершается.
   2. Никто больше не предлагает бесконечное число раз -> `p` становится единственным активным proposer'ом -> его следующее предложение принимается  кворумом без единого NACK -> `p` сам завершается.

В обоих случаях p завершается –--> получаем противоречие.

---
## 6. Что сломается при удалении сообщений LEARN
Без LEARN убирается наблюдаемость результата: значение технически выбирается так же корректно, как и раньше, но узнать о нём в общем случае может `только тот процесс, который сам успешно завершил Propose`. Пропадает возможность для остальных процессов своевременно получать актуальное значение через reliable broadcast (send [LEARN, v'] to all): каждый, кто принимал LEARN ,рассылал его всем. Без RB пропадает гарантия, что значение в принципе дойдёт до всех корректных процессов.

---
## 7. Адаптация к Byzantine отказам, нужен ли PKI
TBD
