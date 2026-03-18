
# IR

## Задача

Дана программа на языке Си (в соответствии с Вашим вариантом). Требуется:
1. Описать программу
2. Построить граф потока управления, дерево доминаторов, граф фронта доминирования, промежуточное представление Generic IR из лекций в SSA форме для данной
функции.
3. Предложить оптимизации, которые можно применить к данной программе. Последовательно применить эти оптимизации к программе и построить промежуточное представление после применения каждой из них.

## Программа

Представленная программа на C:
```C
int function2 (int ∗a , int n) {
    int result = 0;
    for (int i = 0 ; i < n ; ++i) {
        int x = 2;
        int y = 1;

        if (i % 2 == 0) {
            result += a[i] * x * y
        } else {
            result -= a[i] + x * y
        }
    }
    return result;
}
```

Для представления в трёхадресной форме, сначала представим цикл `for` в виде `do-while`

Из цикла:
```C
for (init-statement; condition; increment) {
    loop-body;
}
```
Получается:
```C
init-statement;
if (!condition)
    return;
do {
    loop-body;
    increment;
} while(condition);
```

Эквивалентная программа через `do-while`:

```C
0:   int function2 (int∗ a , int n) {
1:      int result = 0;
2:      int i = 0;
3:      if (!(i < n))
4:         return result;
5:      do {
6:          int x = 2;
7:          int y = 1;
8:          if (i % 2 == 0) {
9:              result += a[i] * x * y;
10:         } else {
11:             result -= a[i] + x * y;
12:         }
13:         i++;
14:     } while (i < n);
15:     return result;
16: }
```

Разобъём код на логические блоки:
* bb0: строки 1-2 (вход)
* bb1: строки 3 и 14 (проверка условия `i < n`)
* bb2: строки 6-8 (тело цикла)
* bb3: строка 9 (true)
* bb4: строка 11 (false)
* bb5: строка 13 (инкремент и возврат к bb1)

Начальная трёхадресная форма:
```C
int function2 (int∗ a , int n) {
bb0:
    result = 0;
    i = 0;
    br b1;
bb1:
    c0 = i < n;
    br c0, bb2, bb6;
bb2:
    x = 2;
    y = 1;
    t0 = i % 2;
    c1 = t0 == 0;
    br c1, bb3, bb4;
bb3:
    t1 = i * 4;
    t2 = a + t1;
    t3 = *t2;
    t4 = x * y;
    t6 = t3 * t4;
    result = result + t6;
    br bb5;
bb4:
    t7 = i1 * 4;
    t8 = a + t7;
    t9 = *t8;
    t10 = x * y;
    t11 = t9 + t10;
    result = result - t11;
    br bb5;
bb5:
    i2 = i1 + 1;
    br bb1;
bb6:
    return result;
}
```

Чтобы построить оптимальную SSA-форму требуется исследовать граф потока управления.

## Граф потока управления
фото
bb0 -> bb1
bb1 -> bb2, bb1 -> bb6
bb2 -> bb3, bb2 -> bb4
bb3 -> bb5, bb4 -> bb5
bb5 -> bb1

## Дерево доминаторов
фото
bb0 -> bb1
bb1 -> bb2, bb1 -> bb6
bb2 -> bb3, bb2 -> bb4, bb2 -> bb5

Непосредственные доминаторы:
```C
IDom(bb1) = bb0
IDom(bb2) = bb1
IDom(bb3) = bb2,
IDom(bb4) = bb2, 
IDom(bb5) = bb2
IDom(bb6) = bb1
```

## Фронт доминирования

При построении фронта доминирования имеет смысл рассматривать только базовые блоки с несколькими предшественниками, так как другие ноды не будут пополнять множества доминирования. Рассмотрим bb0 (как первый), bb1 (предшественники bb0 и bb5), bb5(предшественники bb3 и bb4)
1. bb0
    Нет предшественников $\Rightarrow$ пропускаем
    DF(bb0) = {$\emptyset$}
2. bb1
    1) bb0: IDom(bb1) = bb0 $\Rightarrow$ пропускаем
    2) bb5:
        * Добавляем bb1 в DF(bb5) 
        * IDom(bb5) = bb2 $\Rightarrow$ добавляем bb1 в DF(bb2)
        * IDom(bb2) = bb1 $\Rightarrow$ добавляем bb1 в DF(bb1)
        * IDom(bb1) = bb0 = = IDom(bb1) $\Rightarrow$ стоп
3. bb5
    1) bb3
        * Добавляем bb5 в DF(bb3)
        * IDom(bb3) = bb2 = IDom(bb5) $\Rightarrow$ стоп
    2) bb4
        * Добавляем bb5 в DF(bb4)
        * IDom(bb4) = bb2 = IDom(bb5) $\Rightarrow$ стоп
Итого:

DF(bb0) = $\emptyset$
DF(bb1) = {bb1}
DF(bb2) = {bb1}
DF(bb3) = {bb5}
DF(bb4) = {bb5}
DF(bb5) = {bb1}
DF(bb6) = $\emptyset$

Оптимально разместим $\phi$-функций 
1. result: bb0, bb3, bb4 
    DF(bb4) = DF(bb3) = {bb5} $\Rightarrow$ вставляются res1 = phi(res0, res3) в bb1, так как DF(bb5) = {bb1}, и res4 = phi(res2, res3) в bb5.

2. i: bb0, bb5.
    DF(bb5) = {bb1} $\Rightarrow$ вставляется i1 = phi(i0, i2) в bb1.
    
Приведём итоговый IR в оптимальной SSA-форме:

```C
int function2 (int∗ a , int n) {
bb0:
    res0 = 0;
    i0 = 0;
    br b1;
bb1:
    res1 = phi(res0, res4);
    i1 = phi(i0, i2);
    c0 = i1 < n;
    br c0, bb2, bb6;
bb2:
    x = 2;
    y = 1;
    t0 = i1 % 2;
    c1 = t0 == 0;
    br c1, bb3, bb4;
bb3:
    t1 = i * 4;
    t2 = a + t1;
    t3 = *t2;
    t4 = x * y;
    t6 = t3 * t4;
    res2 = res1 + t6;
    br bb5;
bb4:
    t7 = i1 * 4;
    t8 = a + t7;
    t9 = *t8;
    t10 = x * y;
    t11 = t9 + t10;
    res3 = res1 - t11;
    br bb5;
bb5:
    res4 = phi(res2, res3)
    i2 = i1 + 1;
    br bb1;
bb6:
    return res;
}
```

