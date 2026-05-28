# Лабораторная работа 
## Построение бинарного дерева

### Цель работы

Разработать программу на языке Python, которая рекурсивно строит бинарное дерево заданной высоты, начиная с указанного корня. Реализовать возможность представления дерева в виде **словаря** (базовый вариант) или **списка** (исследовательский вариант). Изучить различные структуры данных для хранения дерева, включая контейнеры из модуля `collections`.

### Индивидуальное задание

В соответствии с вариантом (номер в группе – 11) заданы следующие параметры:

| Параметр       | Значение |
|----------------|----------|
| `root`         | 11       |
| `height`       | 3        |
| Левый потомок  | `root ** 2` |
| Правый потомок | `2 + root ** 2` |

Дерево должно обладать свойством: каждый узел имеет ровно двух потомков (левый и правый). Если высота `height <= 0`, узел отсутствует (возвращается `None`).

### Реализация
```python
"""
Модуль для генерации бинарного дерева различными способами.

Описание:
    Модуль содержит реализацию функции `gen_bin_tree`, которая строит бинарное дерево
    заданной высоты, начиная с указанного корня. Каждый узел имеет два потомка:
    левый и правый. Структура может быть представлена как словарь (по умолчанию)
    или как список.
"""

from typing import Union, Dict, List, Optional


def gen_bin_tree(height: int = 3, root: int = 11, as_list: bool = False) -> Optional[Union[Dict, List]]:
    """
    Рекурсивная генерация бинарного дерева.

    Args:
        height (int): Высота дерева. Определяет количество уровней (включая корень).
        root (int): Значение в корневом узле.
        as_list (bool): Если True — возвращает дерево в виде вложенных списков,
        иначе — в виде словаря (по умолчанию).

    Returns:
        Optional[Union[Dict, List]]: Структура бинарного дерева в виде списка. 
        Если height <= 0, возвращает None.

    """
    if height <= 0:
        return None

    left_val = root ** 2
    right_val = 2 + root ** 2

    # рекурсивное построение потомков
    left_child = gen_bin_tree(height - 1, left_val, as_list)
    right_child = gen_bin_tree(height - 1, right_val, as_list)

    if as_list:
        return [root, left_child, right_child]
    else:
        return {"value": root, "left": left_child, "right": right_child}


if __name__ == "__main__":
    # пример использования
    from pprint import pprint

    print("\nБинарное дерево в виде списка:")
    pprint(gen_bin_tree(3, 11, as_list=True))
```
### Тестирование
Для проверки корректности работы написаны тесты с использованием встроенного модуля unittest. Они проверяют:

* базовую структуру дерева заданной высоты;

* правильность вычисления значений потомков;

* обработку граничных условий (height <= 0);

* соответствие типов возвращаемых данных.
```python
import unittest

class TestBinTree(unittest.TestCase):

    def test_height_zero_returns_none(self):
        self.assertIsNone(gen_bin_tree(height=0, root=10))
        self.assertIsNone(gen_bin_tree(height=-1, root=5))

    def test_height_one_dict(self):
        tree = gen_bin_tree(height=1, root=11, as_list=False)
        expected = {"value": 11, "left": None, "right": None}
        self.assertEqual(tree, expected)

    def test_height_one_list(self):
        tree = gen_bin_tree(height=1, root=11, as_list=True)
        self.assertEqual(tree, [11, None, None])

    def test_custom_values_dict(self):
        # Для root=11, left_val=121, right_val=123
        tree = gen_bin_tree(height=2, root=11, as_list=False)
        self.assertEqual(tree["value"], 11)
        self.assertEqual(tree["left"]["value"], 121)
        self.assertEqual(tree["right"]["value"], 123)
        self.assertIsNone(tree["left"]["left"])
        self.assertIsNone(tree["left"]["right"])
        self.assertIsNone(tree["right"]["left"])
        self.assertIsNone(tree["right"]["right"])

    def test_custom_values_list(self):
        tree = gen_bin_tree(height=2, root=11, as_list=True)
        self.assertEqual(tree[0], 11)
        self.assertEqual(tree[1][0], 121)
        self.assertEqual(tree[2][0], 123)
        self.assertIsNone(tree[1][1])
        self.assertIsNone(tree[1][2])
        self.assertIsNone(tree[2][1])
        self.assertIsNone(tree[2][2])

    def test_consistent_height(self):
        height = 3
        tree = gen_bin_tree(height=height, root=11, as_list=False)
        # Простейшая проверка: количество уровней (рекурсивно обходим)
        def count_levels(node, level=1):
            if node is None:
                return level - 1
            if isinstance(node, dict):
                return max(count_levels(node.get("left"), level+1),
                           count_levels(node.get("right"), level+1))
            return level
        levels = count_levels(tree)
        self.assertEqual(levels, height)

if __name__ == "__main__":
    unittest.main(argv=[''], verbosity=2, exit=False)
```
## Выводы
В ходе работы была реализована рекурсивная функция построения бинарного дерева с настраиваемыми формулами для левого и правого потомков. Проведено тестирование с использованием unittest, все проверки пройдены. Исследованы альтернативные структуры хранения (словарь, список, namedtuple).