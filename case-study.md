# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго,
и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Ассимптотика роста времени работы в зависимости от объема данных

На реальном объеме данных 3250940 строк скрипт выполняется неизвестно сколько времени (не удалось дождаться)

10_000 - 3 сек

20_000 - 14 сек

40_000 - 58 сек

80_000 - 236 cек

Воспользовавшись инструментами, которые умеют по точкам определять зависимость,
можно сделать что зависимость времени от объема данных - некая кубическая.

## Формирование метрики
Пока нет понимания насколько это можно улучшить, поэтому буду стараться уменьшать сложность алгоритма до линейной,
основываясь на поиске точек роста.
Но, говорят, за 30 секунд должно отработать в идеале.

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста в фидбек-лупе позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности сделанных изменений за *время, которое у вас получилось*

Вот как я построил `feedback_loop`:

1) Профилирование алгоритма на объеме данных и поиск точек роста - 20000 строк
2) Улучшение алгоритма в точках роста
3) Тестирование корректности работы программы (task-1-test.rb)
4) Измерение времени работы программы на объеме - 20000 строк


## Вникаем в детали системы, чтобы найти главные точки роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался профилировщиком ruby-prof, stackprof
в различных режимах отображения результатов.

Вот какие проблемы удалось найти и решить

### Ваша находка №1

94% процента времени занимает выполнение поиска select в строке:

`user_sessions = sessions.select { |session| session['user_id'] == user['id'] }`

Решено отказаться от хранения сессий в массиве, а хранить сессии в хэше с ключом user_id,
в значениях же оставить объект сессии без изменений. Тогда доступ ко всем сессиям можно будет получать
`sessions.values.flatten`

Время выполнения программы на 20_000 строках уменьшилось до 0.5 сек

В дальнейшем решено использовать файл в 160_000 строк для оценки скорости работы.

### Ваша находка №2

Следущий результат профилирования в stackprof показал что 36% времени занимает выполнение метода
`collect_stats_from_user`

Не совсем понятно, где именно расходуется время, поэтому пришлось вынести генерацию отчета в отдельный класс
и разбить на методы.

После этого стало понятно, что 55% в генерации отчета занимает сбор статистики по пользователю,
а имеено есть проблема с датами сессий в обратном порядке
`{ 'dates' => user.sessions.map{|s| s['date']}.map {|d| Date.parse(d)}.sort.reverse.map { |d| d.iso8601 } }
`

Избавились от Date.parse в пользу Date.strptime в результате сбор статистики по пользователю стал занимать 49% времени

В результате оптимизации benchmark на 160_000 строках уменьшился с 5.3 до 4.8 сек

### Ваша находка №3

Также профилировщик показывает, что около 28% времени занимает подсчет количества уникальных браузеров.

В результате benchmark на 160_000 строках уменьшился с 4.8 до 3.9 сек

Продолжим вынесение компонентов отчета в отдельные методы.

### Оптимизации неявных моментов

После разбиения программы методы в разных местах программы удалось переписать несколько неоптимальных цепочек map, которые можно выполнить
в одном map.

Гигантский прирост дала оптимизация добавления в массив users_objects после замены `users_objects = users_objects + [user_object]`
на `users_objects << user_object`, что довольно неожиданно для меня.

В результате benchmark на 160_000 строках уменьшился с 3.9 до 2.6 сек

Еще более неожиданным стало что эти улучшения на всем объеме данных позволили уменьшить время выполнения программы
до 92 сек (до этого было 20 минут). Хотя на объеме 160_000 ускорение всего в полтора раза.

Есть мысль, что профилирование по CPU не всегда отражает реальную картину.

### Оптимизация проходов по user_objects

В реузльтате всех улучшений сбор данных по пользователям `usersStats` остался самым тяжелым местом и занимал 55%
выполенния программы.
Были опитимизированы проходы по пользователям, которые выполнялись 7 раз вместо одного.

В результате benchmark на 160_000 строках уменьшился с 2.6 до 1.7 сек

А время выполнения на всех данных уменьшилось до 57 сек

### Оптимизация split при чтении файла

Немного удалось уменьшить время чтения файл, удалив повторные split

В результате benchmark на 160_000 строках уменьшился с 1.7 до 1.5 сек

### Разное

Проведены различные оптимизации: подсчет некоторой статистики в момент загрузки файлов, отказ от конвертации даты
так как в этом нет необходимости, удалось избавиться от user_objects, игнорировать ненужные поля,
избавиться от ненужных проверок.

В результате benchmark на 160_000 строках уменьшился с 1.5 до 1 сек

### Ускорение записи на диск

В итоге профилировщики показали, что 23% времени занимает сохранение файла на диск,
что тоже получилось ускорить, воспользовавшись гемом Oj. Время записи стало 1,5%

В результате benchmark на 160_000 строках уменьшился с 1 до 0.75 сек
На всем объеме время уменьшилось в среднем до 25 секунд

## Результаты
В результате проделанной оптимизации наконец удалось обработать файл с данными.
Получилось улучшить метрику системы с неизвестного количестав времени до 25 секунд и уложиться в заданный бюджет.
Улучшенный скрипт в файле task-1-improved.rb

## Защита от регрессии производительности
Для защиты от потери достигнутого прогресса при дальнейших изменениях программы
добавлен тест на выполнение файла в 160000 строк за 1,2 сек, и тест на линейную ассимптотику.
См task-assert-performance.rb
