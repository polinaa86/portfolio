## Файл readme.md
# Лабораторная работа 2: Паттерн «Декоратор»

## Задание
Реализовать паттерн **Decorator** для получения курсов валют ЦБ РФ:
- **Базовый компонент** — данные в формате **JSON** (API ЦБ).
- **Декораторы** — преобразование в **YAML** (PyYAML) и **CSV** (встроенная библиотека).
- Каждый декоратор имеет метод `operation()` (возвращает данные) и `save_to_file()` (сохраняет в файл).

## Структура
- `decorator.py` — реализация паттерна  
- `test.py` — тесты (unittest)  
- `requirements.txt` — зависимости  

## Установка и запуск
```bash
pip install -r requirements.txt
python decorator.py          # демонстрация работы
python -m unittest test.py -v # запуск тестов
```

## Структура проекта
lab_decorator/
├── .gitignore
├── decorator.py # Основная реализация паттерна
├── test.py # Модульные тесты (unittest)
├── requirements.txt # Зависимости (PyYAML)
├── readme.md # Краткое описание
├── rates.csv # Пример выходного CSV-файла
└── rates.yaml # Пример выходного YAML-файла


## Код программы

### Файл `.gitignore`

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
share/python-wheels/
*.egg-info/
.installed.cfg
*.egg
MANIFEST

# Virtualenv
.venv
venv/
ENV/
env/
.env

# IDE
.vscode/
.idea/
*.swp
*.swo
```

## Файл decorator.py 
```python
"""Модуль реализует паттерн Decorator для форматирования курсов валют ЦБ РФ."""
from __future__ import annotations

import urllib.request
import json
import yaml
import csv
import io
from abc import ABC, abstractmethod
from typing import Any


class Component(ABC):
    """Базовый интерфейс Компонента определяет поведение,
    которое изменяется декораторами."""

    @abstractmethod
    def operation(self) -> str:
        """Возвращает курсы валют в заданном формате."""
        pass


class ConcreteComponent(Component):
    """Конкретный компонент, возвращающий данные в формате JSON
    с помощью API Центробанка РФ."""

    def operation(self) -> str:
        """Получает актуальные курсы валют и возвращает их в виде JSON-строки.

        Returns:
            str: JSON-строка с курсами валют.
        """
        url = "https://www.cbr-xml-daily.ru/daily_json.js"
        with urllib.request.urlopen(url) as response:
            data: dict[str, Any] = json.loads(response.read().decode("utf-8"))
        return json.dumps(data, ensure_ascii=False, indent=2)


class Decorator(Component):
    """Базовый класс Декоратора. Следует интерфейсу Component
    и делегирует работу обёрнутому компоненту."""

    def __init__(self, component: Component) -> None:
        self._component: Component = component

    def operation(self) -> str:
        """Делегирует вызов обёрнутому компоненту."""
        return self._component.operation()


class YamlDecorator(Decorator):
    """Декоратор, преобразующий результат базового компонента в YAML-формат."""

    def operation(self) -> str:
        """Преобразует JSON-результат в YAML-строку.

        Returns:
            str: Данные в формате YAML.
        """
        json_str = self._component.operation()
        data: dict[str, Any] = json.loads(json_str)
        return yaml.dump(
            data,
            allow_unicode=True,
            default_flow_style=False,
            sort_keys=False,
        )

    def save_to_file(self, filename: str = "rates.yaml") -> None:
        """Сохраняет данные в YAML-файл.

        Args:
            filename (str): Имя файла для сохранения.
        """
        with open(filename, "w", encoding="utf-8") as f:
            f.write(self.operation())


class CsvDecorator(Decorator):
    """Декоратор, преобразующий результат базового компонента в CSV-формат."""

    def operation(self) -> str:
        """Преобразует JSON-результат в CSV-строку.

        Returns:
            str: Данные в формате CSV (с шапкой и датой).
        """
        json_str = self._component.operation()
        data: dict[str, Any] = json.loads(json_str)

        output = io.StringIO(newline="")
        writer = csv.writer(output)

        # Добавляем дату
        writer.writerow(["Дата:", data.get("Date", "N/A")])
        writer.writerow([])

        # Шапка таблицы
        writer.writerow(["CharCode", "NumCode", "Nominal", "Name", "Value", "Previous"])

        for valute in data.get("Valute", {}).values():
            writer.writerow(
                [
                    valute.get("CharCode", ""),
                    valute.get("NumCode", ""),
                    valute.get("Nominal", ""),
                    valute.get("Name", ""),
                    valute.get("Value", ""),
                    valute.get("Previous", ""),
                ]
            )

        return output.getvalue()

    def save_to_file(self, filename: str = "rates.csv") -> None:
        """Сохраняет данные в CSV-файл.

        Args:
            filename (str): Имя файла для сохранения.
        """
        with open(filename, "w", encoding="utf-8", newline="") as f:
            f.write(self.operation())


if __name__ == "__main__":
    print("Клиент: Получаем базовый JSON")
    simple = ConcreteComponent()
    print("RESULT (JSON, первые 300 символов):")
    print(simple.operation()[:300] + "...\n")

    print("Клиент: YAML-декоратор")
    yaml_dec = YamlDecorator(simple)
    print("RESULT (YAML, первые 300 символов):")
    print(yaml_dec.operation()[:300] + "...")
    yaml_dec.save_to_file()
    print(f"✅ Сохранено в rates.yaml\n")

    print("Клиент: CSV-декоратор")
    csv_dec = CsvDecorator(simple)
    print("RESULT (CSV, первые 300 символов):")
    print(csv_dec.operation()[:300] + "...")
    csv_dec.save_to_file()
    print(f"✅ Сохранено в rates.csv")
```
## Файл requirements.txt
```txt
PyYAML
```
## Файл test.py
```python
"""Тесты"""
import unittest
from unittest.mock import patch, MagicMock
import json
import yaml
import os
import tempfile

from decorator import (
    ConcreteComponent,
    YamlDecorator,
    CsvDecorator,
)


class TestCurrencyDecorator(unittest.TestCase):
    """Тесты базового компонента и декораторов (по 2 теста на каждый)."""

    # ====================== 2 теста для ConcreteComponent ======================
    @patch("decorator.urllib.request.urlopen")
    def test_concrete_component_operation(self, mock_urlopen):
        """Тест 1: проверка возврата JSON и структуры данных."""
        mock_response = MagicMock()
        mock_response.read.return_value = (
            b'{"Date":"14.04.2026","Valute":{"USD":{"CharCode":"USD","Value":100.0}}}'
        )
        mock_urlopen.return_value.__enter__.return_value = mock_response

        comp = ConcreteComponent()
        result = comp.operation()

        self.assertIsInstance(result, str)
        data = json.loads(result)
        self.assertIn("Valute", data)
        self.assertIn("USD", data["Valute"])

    @patch("decorator.urllib.request.urlopen")
    def test_concrete_component_pretty_json(self, mock_urlopen):
        """Тест 2: проверка, что JSON красиво отформатирован (indent)."""
        mock_response = MagicMock()
        mock_response.read.return_value = b'{"Date":"14.04.2026"}'
        mock_urlopen.return_value.__enter__.return_value = mock_response

        comp = ConcreteComponent()
        result = comp.operation()
        self.assertIn("\n  ", result)  # indent=2

    # ====================== 2 теста для YamlDecorator ======================
    @patch("decorator.urllib.request.urlopen")
    def test_yaml_decorator_operation(self, mock_urlopen):
        """Тест 1: проверка преобразования в YAML."""
        mock_response = MagicMock()
        mock_response.read.return_value = (
            b'{"Date":"14.04.2026","Valute":{"USD":{"CharCode":"USD"}}}'
        )
        mock_urlopen.return_value.__enter__.return_value = mock_response

        base = ConcreteComponent()
        dec = YamlDecorator(base)
        result = dec.operation()

        self.assertIsInstance(result, str)
        data = yaml.safe_load(result)
        self.assertEqual(data["Date"], "14.04.2026")

    @patch("decorator.urllib.request.urlopen")
    def test_yaml_decorator_save(self, mock_urlopen):
        """Тест 2: проверка сохранения в YAML-файл."""
        mock_response = MagicMock()
        mock_response.read.return_value = b'{"Date":"14.04.2026"}'
        mock_urlopen.return_value.__enter__.return_value = mock_response

        base = ConcreteComponent()
        dec = YamlDecorator(base)

        with tempfile.TemporaryDirectory() as tmp:
            fname = os.path.join(tmp, "test.yaml")
            dec.save_to_file(fname)
            self.assertTrue(os.path.exists(fname))

            with open(fname, encoding="utf-8") as f:
                content = f.read()
            self.assertIn("Date:", content)

    # ====================== 2 теста для CsvDecorator ======================
    @patch("decorator.urllib.request.urlopen")
    def test_csv_decorator_operation(self, mock_urlopen):
        """Тест 1: проверка преобразования в CSV."""
        mock_response = MagicMock()
        mock_response.read.return_value = (
            b'{"Date":"14.04.2026","Valute":{"USD":{"CharCode":"USD","Value":100}}}'
        )
        mock_urlopen.return_value.__enter__.return_value = mock_response

        base = ConcreteComponent()
        dec = CsvDecorator(base)
        result = dec.operation()

        self.assertIsInstance(result, str)
        self.assertIn("Дата:", result)
        self.assertIn("USD", result)

    @patch("decorator.urllib.request.urlopen")
    def test_csv_decorator_save(self, mock_urlopen):
        """Тест 2: проверка сохранения в CSV-файл."""
        mock_response = MagicMock()
        mock_response.read.return_value = b'{"Date":"14.04.2026"}'
        mock_urlopen.return_value.__enter__.return_value = mock_response

        base = ConcreteComponent()
        dec = CsvDecorator(base)

        with tempfile.TemporaryDirectory() as tmp:
            fname = os.path.join(tmp, "test.csv")
            dec.save_to_file(fname)
            self.assertTrue(os.path.exists(fname))

            with open(fname, encoding="utf-8") as f:
                content = f.read()
            self.assertIn("Дата:", content)


if __name__ == "__main__":
    unittest.main(verbosity=2)
```
## Файл rates.csv
```csv
Дата:,2026-04-15T11:30:00+03:00

CharCode,NumCode,Nominal,Name,Value,Previous
AUD,036,1,Австралийский доллар,53.7268,53.6792
AZN,944,1,Азербайджанский манат,44.6195,44.8523
DZD,012,100,Алжирских динаров,57.41,57.6784
GBP,826,1,Фунт стерлингов,102.106,102.6386
AMD,051,100,Армянских драмов,20.2173,20.3227
BHD,048,1,Бахрейнский динар,201.6937,202.7458
BYN,933,1,Белорусский рубль,26.7985,26.835
BOB,068,1,Боливиано,10.9773,11.0346
BRL,986,1,Бразильский реал,15.0979,15.1812
HUF,348,100,Форинтов,24.5989,24.3234
VND,704,10000,Донгов,30.2132,30.3708
HKD,344,10,Гонконгских долларов,97.0114,97.5299
GEL,981,1,Лари,28.1302,28.277
DKK,208,1,Датская крона,11.86,11.9496
AED,784,1,Дирхам ОАЭ,20.6544,20.7621
USD,840,1,Доллар США,75.8532,76.2489
EUR,978,1,Евро,89.2556,89.1373
EGP,818,10,Египетских фунтов,14.4459,14.3487
INR,356,100,Индийских рупий,81.2408,81.6646
IDR,360,10000,Рупий,44.3016,44.5587
IRR,364,1000000,Иранских риалов,54.3491,54.6326
KZT,398,100,Тенге,15.9725,16.1158
CAD,124,1,Канадский доллар,54.9183,55.1529
QAR,634,1,Катарский риал,20.8388,20.9475
KGS,417,100,Сомов,86.7389,87.1914
CNY,156,1,Юань,11.091,11.1338
CUP,192,10,Кубинских песо,31.6055,31.7704
MDL,498,10,Молдавских леев,44.1631,44.3935
MNT,496,1000,Тугриков,21.228,21.3398
NGN,566,1000,Найр,55.9313,56.1939
NZD,554,1,Новозеландский доллар,44.4348,44.3654
NOK,578,10,Норвежских крон,79.8581,80.3263
OMR,512,1,Оманский риал,197.2775,198.3066
PLN,985,1,Злотый,21.0616,20.9625
SAR,682,1,Саудовский риял,20.2275,20.333
RON,946,1,Румынский лей,17.5602,17.4767
XDR,960,1,СДР (специальные права заимствования),103.8506,104.4717
SGD,702,1,Сингапурский доллар,59.5768,59.7515
TJS,972,10,Сомони,79.843,80.1903
THB,764,10,Батов,23.5774,23.7004
BDT,050,100,Так,61.7277,62.0497
TRY,949,10,Турецких лир,16.9786,17.1131
TMT,934,1,Новый туркменский манат,21.6723,21.7854
UZS,860,10000,Узбекских сумов,62.4749,62.88
UAH,980,10,Гривен,17.4376,17.5538
CZK,203,10,Чешских крон,36.7488,36.5509
SEK,752,10,Шведских крон,81.3688,82.4059
CHF,756,1,Швейцарский франк,97.1729,96.4565
ETB,230,100,Эфиопских быров,48.4852,48.5489
RSD,941,100,Сербских динаров,76.0623,75.7829
ZAR,710,10,Рэндов,46.3857,46.1318
KRW,410,1000,Вон,50.915,51.5265
JPY,392,100,Иен,47.6435,47.7003
MMK,104,1000,Кьятов,36.1206,36.309
```
## Файл rates.yaml
```yaml
Date: '2026-04-15T11:30:00+03:00'
PreviousDate: '2026-04-14T11:30:00+03:00'
PreviousURL: //www.cbr-xml-daily.ru/archive/2026/04/14/daily_json.js
Timestamp: '2026-04-14T20:00:00+03:00'
Valute:
  AUD:
    ID: R01010
    NumCode: '036'
    CharCode: AUD
    Nominal: 1
    Name: Австралийский доллар
    Value: 53.7268
    Previous: 53.6792
  AZN:
    ID: R01020A
    NumCode: '944'
    CharCode: AZN
    Nominal: 1
    Name: Азербайджанский манат
    Value: 44.6195
    Previous: 44.8523
  DZD:
    ID: R01030
    NumCode: '012'
    CharCode: DZD
    Nominal: 100
    Name: Алжирских динаров
    Value: 57.41
    Previous: 57.6784
  GBP:
    ID: R01035
    NumCode: '826'
    CharCode: GBP
    Nominal: 1
    Name: Фунт стерлингов
    Value: 102.106
    Previous: 102.6386
  AMD:
    ID: R01060
    NumCode: '051'
    CharCode: AMD
    Nominal: 100
    Name: Армянских драмов
    Value: 20.2173
    Previous: 20.3227
  BHD:
    ID: R01080
    NumCode: 048
    CharCode: BHD
    Nominal: 1
    Name: Бахрейнский динар
    Value: 201.6937
    Previous: 202.7458
  BYN:
    ID: R01090B
    NumCode: '933'
    CharCode: BYN
    Nominal: 1
    Name: Белорусский рубль
    Value: 26.7985
    Previous: 26.835
  BOB:
    ID: R01105
    NumCode: 068
    CharCode: BOB
    Nominal: 1
    Name: Боливиано
    Value: 10.9773
    Previous: 11.0346
  BRL:
    ID: R01115
    NumCode: '986'
    CharCode: BRL
    Nominal: 1
    Name: Бразильский реал
    Value: 15.0979
    Previous: 15.1812
  HUF:
    ID: R01135
    NumCode: '348'
    CharCode: HUF
    Nominal: 100
    Name: Форинтов
    Value: 24.5989
    Previous: 24.3234
  VND:
    ID: R01150
    NumCode: '704'
    CharCode: VND
    Nominal: 10000
    Name: Донгов
    Value: 30.2132
    Previous: 30.3708
  HKD:
    ID: R01200
    NumCode: '344'
    CharCode: HKD
    Nominal: 10
    Name: Гонконгских долларов
    Value: 97.0114
    Previous: 97.5299
  GEL:
    ID: R01210
    NumCode: '981'
    CharCode: GEL
    Nominal: 1
    Name: Лари
    Value: 28.1302
    Previous: 28.277
  DKK:
    ID: R01215
    NumCode: '208'
    CharCode: DKK
    Nominal: 1
    Name: Датская крона
    Value: 11.86
    Previous: 11.9496
  AED:
    ID: R01230
    NumCode: '784'
    CharCode: AED
    Nominal: 1
    Name: Дирхам ОАЭ
    Value: 20.6544
    Previous: 20.7621
  USD:
    ID: R01235
    NumCode: '840'
    CharCode: USD
    Nominal: 1
    Name: Доллар США
    Value: 75.8532
    Previous: 76.2489
  EUR:
    ID: R01239
    NumCode: '978'
    CharCode: EUR
    Nominal: 1
    Name: Евро
    Value: 89.2556
    Previous: 89.1373
  EGP:
    ID: R01240
    NumCode: '818'
    CharCode: EGP
    Nominal: 10
    Name: Египетских фунтов
    Value: 14.4459
    Previous: 14.3487
  INR:
    ID: R01270
    NumCode: '356'
    CharCode: INR
    Nominal: 100
    Name: Индийских рупий
    Value: 81.2408
    Previous: 81.6646
  IDR:
    ID: R01280
    NumCode: '360'
    CharCode: IDR
    Nominal: 10000
    Name: Рупий
    Value: 44.3016
    Previous: 44.5587
  IRR:
    ID: R01300
    NumCode: '364'
    CharCode: IRR
    Nominal: 1000000
    Name: Иранских риалов
    Value: 54.3491
    Previous: 54.6326
  KZT:
    ID: R01335
    NumCode: '398'
    CharCode: KZT
    Nominal: 100
    Name: Тенге
    Value: 15.9725
    Previous: 16.1158
  CAD:
    ID: R01350
    NumCode: '124'
    CharCode: CAD
    Nominal: 1
    Name: Канадский доллар
    Value: 54.9183
    Previous: 55.1529
  QAR:
    ID: R01355
    NumCode: '634'
    CharCode: QAR
    Nominal: 1
    Name: Катарский риал
    Value: 20.8388
    Previous: 20.9475
  KGS:
    ID: R01370
    NumCode: '417'
    CharCode: KGS
    Nominal: 100
    Name: Сомов
    Value: 86.7389
    Previous: 87.1914
  CNY:
    ID: R01375
    NumCode: '156'
    CharCode: CNY
    Nominal: 1
    Name: Юань
    Value: 11.091
    Previous: 11.1338
  CUP:
    ID: R01395
    NumCode: '192'
    CharCode: CUP
    Nominal: 10
    Name: Кубинских песо
    Value: 31.6055
    Previous: 31.7704
  MDL:
    ID: R01500
    NumCode: '498'
    CharCode: MDL
    Nominal: 10
    Name: Молдавских леев
    Value: 44.1631
    Previous: 44.3935
  MNT:
    ID: R01503
    NumCode: '496'
    CharCode: MNT
    Nominal: 1000
    Name: Тугриков
    Value: 21.228
    Previous: 21.3398
  NGN:
    ID: R01520
    NumCode: '566'
    CharCode: NGN
    Nominal: 1000
    Name: Найр
    Value: 55.9313
    Previous: 56.1939
  NZD:
    ID: R01530
    NumCode: '554'
    CharCode: NZD
    Nominal: 1
    Name: Новозеландский доллар
    Value: 44.4348
    Previous: 44.3654
  NOK:
    ID: R01535
    NumCode: '578'
    CharCode: NOK
    Nominal: 10
    Name: Норвежских крон
    Value: 79.8581
    Previous: 80.3263
  OMR:
    ID: R01540
    NumCode: '512'
    CharCode: OMR
    Nominal: 1
    Name: Оманский риал
    Value: 197.2775
    Previous: 198.3066
  PLN:
    ID: R01565
    NumCode: '985'
    CharCode: PLN
    Nominal: 1
    Name: Злотый
    Value: 21.0616
    Previous: 20.9625
  SAR:
    ID: R01580
    NumCode: '682'
    CharCode: SAR
    Nominal: 1
    Name: Саудовский риял
    Value: 20.2275
    Previous: 20.333
  RON:
    ID: R01585F
    NumCode: '946'
    CharCode: RON
    Nominal: 1
    Name: Румынский лей
    Value: 17.5602
    Previous: 17.4767
  XDR:
    ID: R01589
    NumCode: '960'
    CharCode: XDR
    Nominal: 1
    Name: СДР (специальные права заимствования)
    Value: 103.8506
    Previous: 104.4717
  SGD:
    ID: R01625
    NumCode: '702'
    CharCode: SGD
    Nominal: 1
    Name: Сингапурский доллар
    Value: 59.5768
    Previous: 59.7515
  TJS:
    ID: R01670
    NumCode: '972'
    CharCode: TJS
    Nominal: 10
    Name: Сомони
    Value: 79.843
    Previous: 80.1903
  THB:
    ID: R01675
    NumCode: '764'
    CharCode: THB
    Nominal: 10
    Name: Батов
    Value: 23.5774
    Previous: 23.7004
  BDT:
    ID: R01685
    NumCode: '050'
    CharCode: BDT
    Nominal: 100
    Name: Так
    Value: 61.7277
    Previous: 62.0497
  TRY:
    ID: R01700J
    NumCode: '949'
    CharCode: TRY
    Nominal: 10
    Name: Турецких лир
    Value: 16.9786
    Previous: 17.1131
  TMT:
    ID: R01710A
    NumCode: '934'
    CharCode: TMT
    Nominal: 1
    Name: Новый туркменский манат
    Value: 21.6723
    Previous: 21.7854
  UZS:
    ID: R01717
    NumCode: '860'
    CharCode: UZS
    Nominal: 10000
    Name: Узбекских сумов
    Value: 62.4749
    Previous: 62.88
  UAH:
    ID: R01720
    NumCode: '980'
    CharCode: UAH
    Nominal: 10
    Name: Гривен
    Value: 17.4376
    Previous: 17.5538
  CZK:
    ID: R01760
    NumCode: '203'
    CharCode: CZK
    Nominal: 10
    Name: Чешских крон
    Value: 36.7488
    Previous: 36.5509
  SEK:
    ID: R01770
    NumCode: '752'
    CharCode: SEK
    Nominal: 10
    Name: Шведских крон
    Value: 81.3688
    Previous: 82.4059
  CHF:
    ID: R01775
    NumCode: '756'
    CharCode: CHF
    Nominal: 1
    Name: Швейцарский франк
    Value: 97.1729
    Previous: 96.4565
  ETB:
    ID: R01800
    NumCode: '230'
    CharCode: ETB
    Nominal: 100
    Name: Эфиопских быров
    Value: 48.4852
    Previous: 48.5489
  RSD:
    ID: R01805F
    NumCode: '941'
    CharCode: RSD
    Nominal: 100
    Name: Сербских динаров
    Value: 76.0623
    Previous: 75.7829
  ZAR:
    ID: R01810
    NumCode: '710'
    CharCode: ZAR
    Nominal: 10
    Name: Рэндов
    Value: 46.3857
    Previous: 46.1318
  KRW:
    ID: R01815
    NumCode: '410'
    CharCode: KRW
    Nominal: 1000
    Name: Вон
    Value: 50.915
    Previous: 51.5265
  JPY:
    ID: R01820
    NumCode: '392'
    CharCode: JPY
    Nominal: 100
    Name: Иен
    Value: 47.6435
    Previous: 47.7003
  MMK:
    ID: R02005
    NumCode: '104'
    CharCode: MMK
    Nominal: 1000
    Name: Кьятов
    Value: 36.1206
    Previous: 36.309
```
### Выводы:
#### В ходе выполнения лабораторной работы:

* Реализован паттерн Декоратор в соответствии с классической структурой (абстрактный компонент, конкретный компонент, абстрактный декоратор, конкретные декораторы).

* Использован ABC и @abstractmethod для создания интерфейса.

* Базовый компонент получает реальные данные из открытого API ЦБ РФ (cbr-xml-daily.ru) и возвращает их в формате JSON.

#### Разработаны два декоратора:

* YamlDecorator – преобразует JSON в YAML (библиотека PyYAML).

* CsvDecorator – преобразует JSON в CSV (встроенный модуль csv).
