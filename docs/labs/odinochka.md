# Лабораторная работа: Работа с валютами. Шаблон «Одиночка»
## Задание
Реализовать получение курсов валют ЦБ РФ в ООП-стиле с использованием **паттерна Singleton** (через метаклассы).  
Проделанные действия:

1. Класс `Currencies` с методами, геттерами/сеттерами, конструктором и деструктором.
2. Значения с плавающей точкой хранятся **отдельно** (целая часть + дробная часть).
3. Формат результата соответствует примеру (с вложенным кортежем `(целая, дробная)`).
4. Номинал валюты сохраняется внутри кэша (чтобы не путаться при переводе в рубли).
5. Контроль частоты запросов (по умолчанию 1 секунда, параметризуется).
6. Тесты: неправильный ID -> `{'R9999': None}`; корректные ID — проверка названия и диапазона 0–999.
7. Метод `visualize_currencies()` получает данные и сохраняет график в `currencies.jpg`.

## Реализация
- **Singleton** реализован через метакласс `SingletonMeta`.
- Запрос к `http://www.cbr.ru/scripts/XML_daily.asp` выполняется **не чаще 1 раза в секунду** (кэширование).
- Парсинг XML + сохранение номинала.
- Визуализация с помощью `matplotlib` (столбчатая диаграмма).

### Запуск
```bash
python main.py
```
### Файл currencies.py
```python
import requests
from xml.etree import ElementTree as ET
import time
import matplotlib.pyplot as plt


class SingletonMeta(type):
    """Метакласс для реализации паттерна Singleton"""
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            instance = super().__call__(*args, **kwargs)
            cls._instances[cls] = instance
        return cls._instances[cls]


class Currencies(metaclass=SingletonMeta):
    """Класс для работы с курсами валют ЦБ РФ (Singleton)"""

    def __init__(self, min_interval: float = 1.0):
        """Конструктор: инициализация атрибутов"""
        self.__min_interval = min_interval          # параметр частоты запросов (секунды)
        self.__last_fetch_time = 0.0                # время последнего успешного запроса
        self.__cached_full = None                   # кэш всех валют (ID -> данные)
        self.__input_params = []                    # входные параметры (список ID)
        self.__result = []                          # результат последнего запроса

    def __del__(self):
        """Деструктор: очистка атрибутов"""
        self.__cached_full = None
        self.__result = []
        self.__input_params = []
        print("Объект Currencies уничтожен (деструктор вызван)")

    # Геттеры и сеттеры
    def get_input_params(self) -> list:
        """Геттер входных параметров"""
        return self.__input_params

    def set_input_params(self, params: list):
        """Сеттер входных параметров"""
        self.__input_params = params

    def get_result(self) -> list:
        """Геттер результата"""
        return self.__result

    def set_result(self, result: list):
        """Сеттер результата"""
        self.__result = result

    def _fetch_currencies(self) -> dict:
        """Приватный метод получения/кэширования всех курсов (с контролем частоты)"""
        current_time = time.time()

        # Если кэш актуален — возвращаем его (запрос НЕ отправляется)
        if (self.__cached_full is not None and
                current_time - self.__last_fetch_time < self.__min_interval):
            print("Используем кэш (запрос слишком частый — интервал соблюден)")
            return self.__cached_full

        # Выполняем запрос (не чаще, чем раз в min_interval)
        try:
            response = requests.get(
                'http://www.cbr.ru/scripts/XML_daily.asp',
                timeout=5
            )
            response.raise_for_status()

            root = ET.fromstring(response.content)
            full_data = {}

            for valute in root.findall("Valute"):
                valute_id = valute.get('ID')
                full_data[valute_id] = {
                    'charcode': valute.find('CharCode').text,
                    'name': valute.find('Name').text,
                    'value': valute.find('Value').text,
                    'nominal': int(valute.find('Nominal').text)  # сохраняем номинал
                }

            self.__cached_full = full_data
            self.__last_fetch_time = current_time
            print(f"Запрос отправлен в ЦБ РФ (новые данные получены, дата: {root.get('Date')})")
            return full_data

        except Exception as e:
            print(f"Ошибка запроса: {e}. Возвращаем старый кэш (если есть)")
            return self.__cached_full or {}

    def get_currencies(self, currencies_ids_lst: list) -> list:
        """Основной метод получения курсов (аналог get_currencies из шаблона)"""
        if not currencies_ids_lst:
            return []

        self.set_input_params(currencies_ids_lst)

        full_data = self._fetch_currencies()
        result = []
        seen = set()

        for rid in currencies_ids_lst:
            if rid in seen:
                continue
            seen.add(rid)

            if rid in full_data:
                info = full_data[rid]
                value_str = info['value']

                # Храним float-значение как отдельно целую и дробную часть
                if ',' in value_str:
                    parts = value_str.split(',')
                    whole = parts[0]
                    frac = parts[1] if len(parts) > 1 else '0'
                else:
                    whole = value_str
                    frac = '0'

                value_tuple = (whole, frac)   # (целая, дробная) — strings для точности

                entry = {
                    info['charcode']: (info['name'], value_tuple)
                }
                result.append(entry)
            else:
                # Тест на неправильный код
                result.append({rid: None})

        self.set_result(result)
        return result

    def visualize_currencies(self):
        """Отдельный метод: получение данных + визуализация"""
        full_data = self._fetch_currencies()   # использует кэш, если запрос слишком частый

        if not full_data:
            print("Нет данных для визуализации")
            return

        # Выбираем популярные валюты для графика (чтобы график был читаемым)
        currencies_to_plot = ['USD', 'EUR', 'GBP', 'CNY', 'JPY', 'TRY', 'KZT', 'CAD', 'AUD']

        plot_data = []
        for info in full_data.values():
            charcode = info['charcode']
            if charcode in currencies_to_plot:
                value_str = info['value'].replace(',', '.')
                try:
                    val = float(value_str)
                    plot_data.append((charcode, val, info['nominal']))
                except ValueError:
                    pass

        if not plot_data:
            print("Не найдено валют для графика")
            return

        # Сортируем по курсу
        plot_data.sort(key=lambda x: x[1], reverse=True)

        labels = [f"{x[0]} (x{x[2]})" for x in plot_data]   # показываем номинал, если ≠1
        values = [x[1] for x in plot_data]

        fig, ax = plt.subplots(figsize=(14, 7))
        bars = ax.bar(labels, values, color='skyblue', edgecolor='navy')

        ax.set_ylabel('Курс в рублях РФ')
        ax.set_title('Курсы валют ЦБ РФ (паттерн Singleton)')
        ax.set_xlabel('Валюта (номинал)')
        plt.xticks(rotation=45, ha='right')

        # Подписи значений над столбцами
        for bar in bars:
            height = bar.get_height()
            ax.text(bar.get_x() + bar.get_width() / 2, height + 0.5,
                    f'{height:.2f}', ha='center', va='bottom', fontsize=9)

        plt.tight_layout()
        plt.savefig('currencies.jpg', dpi=200, bbox_inches='tight')
        plt.close(fig)
        print("График сохранён в currencies.jpg")
```
### Файл main.py
```python
from currencies import Currencies
import time


if __name__ == "__main__":
    print("Запуск лабораторной работы: Работа с валютами (Singleton)\n")

    # Создаём Singleton-объект (нельзя создать больше одного)
    currencies = Currencies(min_interval=1.0)
    print(f"Объект создан: {currencies}")

    ids_example = ['R01035', 'R01335', 'R01700J']
    res = currencies.get_currencies(ids_example)
    print("Результат get_currencies (пример):")
    for item in res:
        print(item)

    print("\n" + "="*60)

    # Тест 1: неправильный код
    wrong_res = currencies.get_currencies(['R9999'])
    print("Тест неправильного ID:")
    print(wrong_res)
    assert len(wrong_res) == 1 and 'R9999' in wrong_res[0] and wrong_res[0]['R9999'] is None, "Тест неправильного ID провален"
    print("Тест неправильного ID пройден")

    # Тест 2: корректный ID (GBP) — название + диапазон
    test_res_gbp = currencies.get_currencies(['R01035'])
    for d in test_res_gbp:
        if 'GBP' in d:
            name, (whole, frac) = d['GBP']
            val_float = float(f"{whole}.{frac}")
            print(f"GBP: название='{name}', курс={val_float}")
            assert "Фунт стерлингов" in name, "Название валюты не русскоязычное"
            assert 0 < val_float < 999, "Курс вне диапазона 0-999"
            print("Тест GBP пройден")
            break

    # Тест 3: корректный ID (USD) — название + диапазон
    test_res_usd = currencies.get_currencies(['R01235'])
    for d in test_res_usd:
        if 'USD' in d:
            name, (whole, frac) = d['USD']
            val_float = float(f"{whole}.{frac}")
            print(f"USD: название='{name}', курс={val_float}")
            assert "Доллар США" in name, "Название валюты не русскоязычное"
            assert 0 < val_float < 999, "Курс вне диапазона 0-999"
            print("Тест USD пройден")
            break

    print("\n" + "="*60)

    # Проверка контроля частоты запросов
    print("Проверка rate-limiting (вызов сразу после предыдущего):")
    res2 = currencies.get_currencies(['R01239'])   # EUR
    print("Второй вызов должен использовать кэш (см. сообщение выше)")

    # Визуализация
    print("\n" + "="*60)
    print("Запуск визуализации...")
    currencies.visualize_currencies()

    print("\nВсе тесты пройдены!")
```

### Выводы
В ходе выполнения лабораторной работы:

* Реализован класс Currencies, строго соответствующий паттерну Singleton через метакласс – гарантированно существует только один экземпляр, что важно для единого кэша и контроля частоты запросов.

* Организовано хранение вещественных чисел курсов в виде отдельной целой и дробной части (строки) – это исключает ошибки округления при использовании float и полностью соответствует заданию.

* Внедрён контроль частоты запросов (rate limiting) с параметризуемым интервалом – при повторных вызовах в течение заданного времени используется кэш, что снижает нагрузку на сервер ЦБ и ускоряет ответ.

* Реализована визуализация данных с помощью библиотеки matplotlib – диаграмма сохраняется как currencies.jpg и может быть встроена в отчёт.
