# Лабораторная работа
## Вычисление деления

## Цель работы

Разработать функцию `calculate`, которая выполняет деление первого операнда на второй с контролем точности результата (epsilon). Написать функцию `load_params`, считывающую значение точности из конфигурационного файла `settings.ini` и передающую его в `calculate`. Покрыть код тестами (позитивные и негативные сценарии).

## Задание

1.  Напишите функцию (calculate), которая принимает на вход 2 операнда, возможно целые,
возможно дробные , выполняет деление операнда 1 на операнд 2
и точность (epsilon, ключевой аргумент) по умолчанию 0.0001
(диапазон: 10**-1 < epsilon < 10**-9)
2. Напишите функцию (load_params), которая считывает значение точности
из конфигурационного файла settings.ini и задает точность для функции 1
3. Напишите тесты для тестирования работы функции 1 (
1/2, epsilon = 0.1 = 0.5, 1/1000 = 0.001 epsilon = 0.001, деление на ноль)
и функции 2 (виды тестов: проверить ситуацию открытия файла на чтение,
epsilon входит в диапазон значений, формат числа в конф. файле )
## Реализация

### Файл `calculator.py`

```python
import math
from typing import Union

Number = Union[int, float]

def calculate(operand1: Number, operand2: Number, epsilon: float = 0.0001) -> float:
    """
    Выполняет деление operand1 / operand2 с заданной точностью epsilon.

    Args:
        operand1 (Number): Делимое (целое или дробное).
        operand2 (Number): Делитель (целое или дробное).
        epsilon (float): Точность (по умолчанию 0.0001). Должна быть в диапазоне (1e-1, 1e-9).

    Returns:
        float: Результат деления, округлённый в соответствии с epsilon.

    Raises:
        ValueError: Если operand2 равен нулю или epsilon вне допустимого диапазона.
    """
    # Проверка деления на ноль
    if operand2 == 0:
        raise ValueError("Деление на ноль невозможно")

    # Проверка диапазона epsilon (10^-1 < epsilon < 10^-9)
    if not (10**-1 > epsilon > 10**-9):
        raise ValueError(f"epsilon = {epsilon} выходит за допустимый диапазон (0.1 > epsilon > 1e-9)")

    # Выполняем деление
    result = operand1 / operand2

    # Определяем количество знаков после запятой на основе порядка epsilon
    # Если epsilon = 0.001, то знаков = 3. Используем логарифм.
    if epsilon > 0:
        decimal_places = max(0, -int(math.floor(math.log10(epsilon))))
        result = round(result, decimal_places)

    return result


def load_params(config_path: str = "settings.ini") -> float:
    """
    Считывает значение epsilon из конфигурационного файла.

    Ожидаемый формат файла: строка вида "epsilon = 0.0001" (пробелы не важны).

    Args:
        config_path (str): Путь к конфигурационному файлу. По умолчанию "settings.ini".

    Returns:
        float: Значение epsilon из файла.

    Raises:
        FileNotFoundError: Если файл не найден.
        ValueError: Если значение отсутствует, не является числом или выходит за диапазон.
    """
    try:
        with open(config_path, 'r', encoding='utf-8') as f:
            content = f.read()
    except FileNotFoundError:
        raise FileNotFoundError(f"Файл {config_path} не найден")

    # Ищем строку, содержащую 'epsilon ='
    lines = content.splitlines()
    epsilon = None
    for line in lines:
        if '=' in line and 'epsilon' in line.lower():
            parts = line.split('=')
            if len(parts) >= 2:
                try:
                    epsilon = float(parts[1].strip())
                    break
                except ValueError:
                    raise ValueError("Некорректный формат числа в конфигурационном файле")

    if epsilon is None:
        raise ValueError("В конфигурационном файле не найдена строка epsilon = ...")

    # Проверяем диапазон (идентично calculate)
    if not (10**-1 > epsilon > 10**-9):
        raise ValueError(f"epsilon = {epsilon} выходит за допустимый диапазон (0.1 > epsilon > 1e-9)")

    return epsilon
```
## Пример конфигурационного файла settings.ini
```ini
epsilon = 0.0001
```
## Файл test_calculator.py (unittest)
```python
import unittest
import os
import tempfile
from calculator import calculate, load_params


class TestCalculate(unittest.TestCase):
    """Тесты для функции calculate"""

    def test_division_1_2_epsilon_0_1(self):
        result = calculate(1, 2, epsilon=0.1)
        # epsilon=0.1 -> округление до 1 знака? 1/2=0.5, округлять не нужно
        self.assertEqual(result, 0.5)

    def test_division_1_1000_epsilon_0_001(self):
        result = calculate(1, 1000, epsilon=0.001)
        # 1/1000 = 0.001, epsilon=0.001 -> 3 знака
        self.assertEqual(result, 0.001)

    def test_division_by_zero(self):
        with self.assertRaises(ValueError) as context:
            calculate(10, 0)
        self.assertIn("Деление на ноль", str(context.exception))

    def test_epsilon_out_of_range_high(self):
        with self.assertRaises(ValueError) as context:
            calculate(1, 2, epsilon=0.5)   # 0.5 > 0.1
        self.assertIn("выходит за допустимый диапазон", str(context.exception))

    def test_epsilon_out_of_range_low(self):
        with self.assertRaises(ValueError) as context:
            calculate(1, 2, epsilon=1e-10)  # 1e-10 < 1e-9
        self.assertIn("выходит за допустимый диапазон", str(context.exception))

    def test_default_epsilon_works(self):
        # Проверка, что с epsilon по умолчанию (0.0001) ошибки нет
        try:
            calculate(1, 3)
        except ValueError:
            self.fail("calculate() с epsilon по умолчанию выбросил исключение")

    def test_float_division_precision(self):
        # 2/3 = 0.666666..., epsilon=0.01 -> округление до 2 знаков: 0.67
        result = calculate(2, 3, epsilon=0.01)
        self.assertAlmostEqual(result, 0.67, places=2)


class TestLoadParams(unittest.TestCase):
    """Тесты для функции load_params"""

    def setUp(self):
        # Создаём временный файл для тестов
        self.temp_dir = tempfile.TemporaryDirectory()
        self.valid_config = os.path.join(self.temp_dir.name, "settings.ini")
        with open(self.valid_config, 'w', encoding='utf-8') as f:
            f.write("epsilon = 0.0001\n")

    def tearDown(self):
        self.temp_dir.cleanup()

    def test_load_valid_epsilon(self):
        eps = load_params(self.valid_config)
        self.assertEqual(eps, 0.0001)

    def test_file_not_found(self):
        with self.assertRaises(FileNotFoundError):
            load_params("non_existent_file.ini")

    def test_epsilon_out_of_range_in_file(self):
        # Создаём файл с недопустимым epsilon
        bad_file = os.path.join(self.temp_dir.name, "bad.ini")
        with open(bad_file, 'w', encoding='utf-8') as f:
            f.write("epsilon = 0.5")
        with self.assertRaises(ValueError) as ctx:
            load_params(bad_file)
        self.assertIn("выходит за допустимый диапазон", str(ctx.exception))

    def test_invalid_number_format(self):
        bad_file = os.path.join(self.temp_dir.name, "invalid.ini")
        with open(bad_file, 'w', encoding='utf-8') as f:
            f.write("epsilon = abc")
        with self.assertRaises(ValueError) as ctx:
            load_params(bad_file)
        self.assertIn("Некорректный формат числа", str(ctx.exception))

    def test_missing_epsilon_line(self):
        bad_file = os.path.join(self.temp_dir.name, "no_epsilon.ini")
        with open(bad_file, 'w', encoding='utf-8') as f:
            f.write("some_other_param = 123")
        with self.assertRaises(ValueError) as ctx:
            load_params(bad_file)
        self.assertIn("не найдена строка epsilon", str(ctx.exception))


if __name__ == "__main__":
    unittest.main(verbosity=2)
```
## Запуск тестов
```bash
python -m unittest test_calculator.py -v
```
## Пример использования
```python
from calculator import calculate, load_params

# Прямой вызов с заданием точности
result = calculate(10, 3, epsilon=0.001)
print(result)  # 3.333 (округлено до трёх знаков)

# Загрузка точности из файла settings.ini
try:
    eps = load_params()
    result2 = calculate(5, 2, epsilon=eps)
    print(f"Результат с epsilon из файла: {result2}")
except (FileNotFoundError, ValueError) as e:
    print(f"Ошибка: {e}")
```
### Выводы
В ходе выполнения лабораторной работы:

* Разработана функция calculate, корректно обрабатывающая деление, округление до заданной точности и проверяющая допустимость epsilon и делителя.

* Реализована функция load_params, читающая конфигурационный файл, проверяющая формат и диапазон значения.

* Написаны тесты, покрывающие как успешные сценарии, так и обработку исключений (деление на ноль, отсутствие файла, неверный формат, выход epsilon за границы).

