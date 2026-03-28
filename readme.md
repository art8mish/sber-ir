
Репозиторий представляет отчёт по заданию курса "Тензорные компиляторы" от Сбер. В данном задании проводится теоритическая компиляция из языка высокого уровня в ассемблер функции:

```C
int function2 (int ∗a , int n) {
    int result = 0;
    for (int i = 0; i < n; ++i) {
        int x = 2;
        int y = 1;

        if (i % 2 == 0)
            result += a[i] * x * y;
        else
            result -= a[i] + x * y;
    }
    return result;
}
```

# Часть 1: IR

Построение Generic IR приводится в файле [ir.md](ir.md)

# Часть 2: Codegen

Процесс кодогенерации описан в файле [codegen.md](codegen.md)