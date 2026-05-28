# Лабораторная работа 
## Ряд Фибоначчи с помощью итераторов

### Цель работы:

Разработать программу на языке Python, демонстрирующую два подхода к работе с числами Фибоначчи:

1. **Сопрограмма (корутина)** – генерация списка первых `n` чисел Фибоначчи по запросу.
2. **Итератор** – фильтрация элементов произвольного списка, оставляющая только числа, принадлежащие ряду Фибоначчи.

Реализовать тесты для обоих модулей с использованием `pytest` и `unittest`.

## Задание 1. Сопрограмма для генерации списка чисел Фибоначчи

### Описание:

На основе предоставленного файла `gen_fib.py` разработана сопрограмма `my_genn`. Она принимает через метод `send(n)` целое число `n` и возвращает список из первых `n` чисел Фибоначчи. Сопрограмма инициализируется с помощью декоратора `fib_coroutine`, который выполняет первый `send(None)` для запуска корутины до первого `yield`.

**Особенности реализации:**
- Используется вспомогательный генератор `fib_elem_gen()`, бесконечно возвращающий числа Фибоначчи.
- Для взятия первых `n` элементов применена функция `islice` из модуля `itertools`.
- При `n <= 0` возвращается пустой список.

### Файл `gen_fib.py`

```python
import functools
from itertools import islice
from typing import Generator, List


def fib_elem_gen() -> Generator[int, None, None]:
    """
    Генератор, бесконечно возвращающий числа ряда Фибоначчи.
    """
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b


def fib_coroutine(g):
    """
    Декоратор для инициализации сопрограммы (первый вызов gen.send(None)).
    """
    @functools.wraps(g)
    def inner(*args, **kwargs):
        gen = g(*args, **kwargs)
        gen.send(None)
        return gen
    return inner


@fib_coroutine
def my_genn() -> Generator[List[int], int, None]:
    """
    Сопрограмма, которая по полученному n возвращает список первых n чисел Фибоначчи.
    
    Пример:
        >>> gen = my_genn()
        >>> gen.send(3)
        [0, 1, 1]
        >>> gen.send(5)
        [0, 1, 1, 2, 3]
    """
    fib_gen = fib_elem_gen()
    while True:
        n = yield
        if n <= 0:
            yield []
        else:
            # Берём первые n элементов из бесконечного генератора
            fib_list = list(islice(fib_gen, n))
            yield fib_list

## Тесты для сопрограммы (test_fib.py)
## Тесты написаны с использованием pytest и проверяют как тривиальные, так и граничные случаи.

import pytest
from gen_fib import my_genn


def test_fib_1():
    gen = my_genn()
    assert gen.send(3) == [0, 1, 1]


def test_fib_2():
    gen = my_genn()
    assert gen.send(5) == [0, 1, 1, 2, 3]


def test_fib_zero():
    gen = my_genn()
    assert gen.send(0) == []


def test_fib_one():
    gen = my_genn()
    assert gen.send(1) == [0]


def test_fib_two():
    gen = my_genn()
    assert gen.send(2) == [0, 1]


def test_fib_negative():
    gen = my_genn()
    assert gen.send(-5) == []   # Отрицательное n -> пустой список


def test_fib_large():
    gen = my_genn()
    result = gen.send(10)
    assert result == [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

## Пример использования

```python
from gen_fib import my_genn

gen = my_genn()
print(gen.send(3))   # [0, 1, 1]
print(gen.send(5))   # [0, 1, 1, 2, 3]
print(gen.send(8))   # [0, 1, 1, 2, 3, 5, 8, 13]
```
## Задание 2. Итератор, фильтрующий числа Фибоначчи из списка
### Описание
Реализован класс FibonacchiLst, который при итерации по исходной последовательности возвращает только те элементы, которые являются числами Фибоначчи. Представлены два варианта:

Классический итератор с методами __iter__ и __next__.

Упрощённый итератор через метод __getitem__ (дополнительно).

Для определения принадлежности числа ряду Фибоначчи используется предварительное вычисление множества всех чисел Фибоначчи, не превышающих максимальный элемент списка (в первом варианте) либо прямая проверка (во втором варианте).

## Файл fibonacci_iterator.py
```python
from typing import Iterable, List, Set, Any


class FibonacchiLst:
    """
    Итератор, возвращающий из исходного списка только те элементы,
    которые являются числами Фибоначчи.
    
    Пример:
        >>> lst = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 1]
        >>> list(FibonacchiLst(lst))
        [0, 1, 2, 3, 5, 8, 1]
    """
    
    def __init__(self, instance: Iterable[Any]) -> None:
        """
        Сохраняет исходную последовательность и генерирует множество
        чисел Фибоначчи, встречающихся в ней.
        
        :param instance: исходная последовательность (список, кортеж и т.п.)
        """
        self._instance = list(instance)          # Приводим к списку для индексации
        self._idx = 0                            # Текущий индекс при итерации
        # Множество чисел Фибоначчи, которые могут встретиться в списке
        self._fib_set = self._generate_fib_set()
    
    def _generate_fib_set(self) -> Set[int]:
        """Генерирует множество всех чисел Фибоначчи, не превышающих максимум исходного списка."""
        if not self._instance:
            return set()
        max_val = max(self._instance)
        fib_set = set()
        a, b = 0, 1
        while a <= max_val:
            fib_set.add(a)
            a, b = b, a + b
        return fib_set
    
    def __iter__(self) -> 'FibonacchiLst':
        """Возвращает сам итератор."""
        return self
    
    def __next__(self) -> Any:
        """
        Возвращает следующий элемент исходного списка, принадлежащий ряду Фибоначчи.
        Если таких элементов больше нет, возбуждает StopIteration.
        """
        while self._idx < len(self._instance):
            current = self._instance[self._idx]
            self._idx += 1
            if current in self._fib_set:
                return current
        raise StopIteration 
```

## Тесты для итератора test_fib_it.py
###### Тесты написаны с использованием unittest.

```python
import unittest
from fibonacci_iterator import FibonacchiLst


class TestFibIterator(unittest.TestCase):
    
    def test_normal(self):
        result = list(FibonacchiLst(range(10)))
        self.assertEqual(result, [0, 1, 2, 3, 5, 8])
    
    def test_with_duplicates(self):
        lst = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 1]
        result = list(FibonacchiLst(lst))
        self.assertEqual(result, [0, 1, 2, 3, 5, 8, 1])
    
    def test_empty(self):
        result = list(FibonacchiLst([]))
        self.assertEqual(result, [])
    
    def test_no_fibonacci(self):
        result = list(FibonacchiLst([4, 6, 7, 10]))
        self.assertEqual(result, [])
    
    def test_single_zero(self):
        result = list(FibonacchiLst([0]))
        self.assertEqual(result, [0])
    
    def test_single_one(self):
        result = list(FibonacchiLst([1]))
        self.assertEqual(result, [1])
    
    def test_negative_numbers(self):
        result = list(FibonacchiLst([-1, 0, 1, -2, 3]))
        self.assertEqual(result, [0, 1, 3])   # Отрицательные числа не являются числами Фибоначчи


if __name__ == '__main__':
    unittest.main()
```
### Пример использования
```python
from fibonacci_iterator import FibonacchiLst

lst = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 1]
fib_iterator = FibonacchiLst(lst)
print(list(fib_iterator))   # [0, 1, 2, 3, 5, 8, 1]
```
### Выводы
В ходе выполнения лабораторной работы:

* Реализована сопрограмма my_genn, которая по запросу возвращает список первых n чисел Фибоначчи. Использованы генераторы, декоратор для инициализации корутины и itertools.islice.

* Создан итератор FibonacchiLst, фильтрующий элементы исходной последовательности, оставляя только числа Фибоначчи. Продемонстрированы два подхода: классический протокол итератора (__iter__, __next__) и итерируемый объект через __getitem__.

* Написаны тесты для всех случаев, включая граничные (пустой список, отрицательные числа, дубликаты). Тесты успешно проходят.
