# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго, и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы я придумал использовать такую метрику: потребляемая память во время выполнения программы.
Первоначальные измерения проводились для модифицированной версии программы из задания 1.
Для обработки файла с 3_250_940 (134 МБ) строками было использовано
RAM for process at the start of process - MEMORY USAGE: 14 MB
RAM for process at the end of process - MEMORY USAGE: 1325 MB
Т.е. в итоге программа использовала 1311 MB памяти

Бюджет метрики - программа не должна потреблять больше 70Мб памяти при обработке файла data_large в течение всей своей работы.

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста в фидбек-лупе позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`.

Вот как я построил `feedback_loop`:
- проверить потребленную память
- настроить и провести профилирование, для профилирования использовать меньший файл, чем полный
- модифицировать код
- проверить тесты
- замерить результаты выполнения

## Вникаем в детали системы, чтобы найти главные точки роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался:
- гем memory_profiler
- ruby-prof в режимах `Flat`, `Graph` и `CallStack`

## Checklist
- [x] Построить и проанализировать отчёт гемом `memory_profiler`
- [x] Построить и проанализировать отчёт `ruby-prof` в режиме `Flat`;
- [x] Построить и проанализировать отчёт `ruby-prof` в режиме `Graph`;
- [x] Построить и проанализировать отчёт `ruby-prof` в режиме `CallStack`;
- [ ] Построить и проанализировать отчёт `ruby-prof` в режиме `CallTree` c визуализацией в `QCachegrind`;
- [ ] Построить и проанализировать текстовый отчёт `stackprof`;
- [ ] Построить и проанализировать отчёт `flamegraph` с помощью `stackprof` и визуализировать его в `speedscope.app`;
- [ ] Построить график потребления памяти в `valgrind massif visualier` и включить скриншот в описание вашего `PR`;
- [x] Написать тест, на то что программа укладывается в бюджет по памяти

### Находка №1
- Первое профилирование при помощи гема memory_profiler, размер файла - 100000 строк
RAM for process at the start of process - MEMORY USAGE: 12 MB
RAM for process at the end of process - MEMORY USAGE: 60 MB (без профилирования)
Total allocated: 101.51 MB (1397362 objects)
Total retained:  0 B (0 objects)

- Основные точки роста
47.38 MB  task-2.rb:22 (split строк)
11.92 MB  task-2.rb:19 (чтение исходного файла и split на строки)
9.10 MB   task-2.rb:39 (сохранение результат)

- Основные точки роста по результатам `ruby-prof`
49% - split строк

- гипотеза улучшения: большой объем памяти занимает чтение файла в память и сохранение его, а также преобразование строк в массив, поэтому действительно надо изменить метод чтения из файла + надо подумать как избежать split

- принятые решения:
  - использовать построчную обработку файла IO.foreach(filename) вместо обработки файла целиком
  - результаты без профилирования
    RAM for process at the end of process - Memory usage: 43.11 MB (для 100000 строк)
    RAM for process at the end of process - Memory usage: 1054.61 MB (для полного файла), экономия 270 MB
  - результаты профилирования
    - основная точка роста 63.27% (73.54%) String#split [100000 calls, 100000 total]

- гипотезы:
  - также хранится весь список юзеров, не имеет смысла, стоит хранить временно только статистику одного юзера, а затем записывать эти данные в файл,
  - также использовать константы для строк и символы для ключей в хеше
  - использовать SortedSet вместо array.uniq.sort, что не показывало результата на скорость из первого задания, но в данном случае массив всех браузеров не будет бесконечно расти, т.е. будет выделено памяти столько, сколько всего уникальных браузеров
  - использовать ! методы

  - результаты без профилирования
    RAM for process at the end of process - Memory usage: 0.31 MB (для 100000 строк)
    RAM for process at the end of process - Memory usage: 1.98 MB (для полного файла), ~ в 660 раз меньше потребляемой памяти
  - результаты профилирования
    - основная точка роста 58.71% (58.71%) String#split [3250940 calls, 3250940 total]
                           20.18% (20.18%) Report#save_not_last_user_to_file [499999 calls, 499999 total]
    - split не удалось как-то уменьшить)
    - но теперь в каждый момент времени максимум в памяти хранится только одна строка с данными, текущее состояние юзера и текущее состояние итогового отчета
  - замер времени выполнения, 32 секунды для полного файла, т.е. потеря нескольких секунд обработки по сравнению с ДЗ 1, но просто громадный скачок в экономии памяти

### Находка №2
- повторная проверка при помощи memory_profiler показала, что тратится
- без профилирования
  Memory usage: 1.2 MB
- memory_profiler
  Total allocated: 90.88 MB (1097606 objects)
  Total retained:  0 B (0 objects)

Основные точки роста
43.38 MB  task-2.rb:29 - split
29.67 MB  task-2.rb:81 - конкатенация строк для статистики юзера

- выдвинул предположение, что если хранить состояние юзера в хэше, то много памяти тратится, теория - хранить в инстансных переменных
- результат
- без профилирования
  Memory usage: 0.47 MB
- memory_profiler
  Total allocated: 78.12 MB (997638 objects)
  Total retained:  0 B (0 objects)

- идея, зачем сохранять весь результат, нужна только часть данных, поэтому cols = line.split(DATA_SPLITTER)[2..5], и если нет 4 элемента, то это user, в итоге меньше сравнений со строками и меньше строк сохраняется в памяти, 10 МБ экономии
- результат
- без профилирования
  Memory usage: 0.2 MB (для 100_000 строк)
  Memory usage: 1.1 MB (для полного файла)
- memory_profiler
  Total allocated: 61.55 MB (1035882 objects)
  Total retained:  0 B (0 objects)

Итог - получается удалось добиться потребления 1.1 MB памяти для обработки полного файла
