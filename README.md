# audio2audio
Скрипт предназначен для автоматического сдвига тайминга субтитров. Т.е. имеется два рипа одного и того же кинца, и субтитры к одному из них. На основе их звукодорожек скрипт должен сопоставить временные метки субтитров моментам второго видео и вернуть четвёртый файл - субтитры ко второму видео, а также вывести информацию о том, в какие местах были произведены сдвиги субтитров: на практике бывают случаи, когда в первом рипе не хватает куска, поэтому это нужно, чтобы можно было просмотреть вручную подозрительные места. Собственно, есть идея вывести ещё и пятый файл - сконструировать из кусков второго видео, отсутствующих в первом, новое видео, которое можно будет просмотреть и проверить, нет ли там новых реплик.
# Формат ввода и вывода
Все данные для запуска скрипта, в частности, путь к медиафайлам и субтитрам, находятся в файле config.py. Надеюсь, по приведённому примеру и комментариям понятно, что каждая из переменных делает. Выводит скрипт информацию о процессе: текущую точность вычисления пути в графе, затраченное время на каждый из путей и т.п. Вывод p(x, y) означает, что поиск в настоящий момент находится в вершине графа с такими координатами, позже переделаю, чтобы выводила процент завершённого вместо этого. Каждый файл субтитров \*.ass, подвергающийся сдвигу, порождает файл вида \*\_shifted.ass в той же директории. 
# Как это вообще запускать?
Нужен Python 3.5 и библиотеки numpy, scipy. Также я не нашёл более простого способа работы с видео и аудио, кроме как запускать команды FFmpeg из питона, так что FFmpeg тоже должен быть установлен и прописан в PATH. Если вы предпочитаете другой способ конвертации видео в аудио данной частоты, пропишите его вместо <b>call('ffmpeg</b> ... в main.py : эта строчка должна вытаскивать из видео <i>%filename%</i> аудиофайл, сохраняя его по расположению <i>%wav_file%</i> с частотой <i>%Config.DEFAULT_HZ%</i>. После прописывания всех данных в конфиге запустите main.py.
# Алгоритм
Разобьём аудиодорожки на много мелких кусочков одинаковой длины, эту длину обзовём <b>тиком</b>. Теперь возьмём все куски (накладывающиеся, разумеется) по <b>B_OVERLAP_DEGREE</b> подряд стоящих тиков (<b>B_OVERLAP_DEGREE</b>=3 в текущей реализации) и подсчитаем спектр для каждого из них. Получим две спектрограммы, которые нужно сопоставить друг другу по похожести. Между любыми двумя спектрами есть некоторое расстояние, подробности см. в коде [функция <b>cos_log</b> из <b>spectrum.py</b>]. Мы хотим, чтобы сопоставлялись куски с как можно меньшим суммарным расстоянием.

Построим граф-решётку, вершинами которого являются пары (спектр номер A из первой спектрограммы, спектр номер B из второй спектрограммы), а рёбра из каждой вершины выходят в трёх направлениях - в (A, B+1) [вертикальное], (A+1, B) [горизонтальное], (A+1, B+1) [диагональное]. Вес диагонального ребра равен расстоянию между спектрами A и B, а горизональных/вертикальных - некоторой константе (<b>NONDIAGKOEF</b>=1.3 в коде) * среднее расстояние между двумя спектрами (предпосчитывается тысячей рандомных подсчётов расстояний и взятием среднего). Теперь мы ищем кратчайший путь из (0, 0) в противоположный конец прямоугольника. В этом пути диагональным участкам будут соответствовать соответствия в видео, а горизонтальным и вертикальным - вырезанные участки. Так как обычно кусков, на которые разбито видео, не очень много, можно ввести довольно приличный штраф за переход с гор./верт. рёбер на диагональные и наоборот. В текущей версии он равен <b>PENALTY</b>=15 * среднее расстояние между двумя спектрами. Предполагаю, что нужно ввести зависимость этой величины не только от среднего расстояния между спектрами, но и от длины спектрограмм, которые будут довольно короткими для больших размеров тика (см. далее).

Проблема в том, что на реальных файлах поиск кратчайшего пути в таком графе работает долго, даже с использованием некоторых оценок снизу на расстояние. Поэтому мы применяем следующую эвристику, довольно стабильно наблюдающуюся: если мы рассмотрим кратчайшие пути для спектрограмм с тиком T и c тиком T/2, то эти пути не будут сильно друг отходить. Поэтому можно взять изначально огромный размер тика, чтобы одна из сторон решётки не превосходила 2 по длине, и при этом был бы равен 2^(нечто) * финальный размер, к которому мы стремимся. После этого можно делить размер тика пополам, ища каждый новый путь в эпсилон-окрестности старого, что делается за линию обычной динамикой. (Для каждой вершины хранятся два лучших пути: заканчивающихся на гор./верт. ребро, и на диагональное ребро, чтобы можно было учитывать штрафы за переходы.)

Заметим, что спектрограмма строится по исходному сигналу только один раз - для размера тика, равному <b>BASE_TICK</b>=0.01 секунды. Спектрограммы для больших размеров тиков получаются усреднениями векторов в этой базовой спектрограмме. А именно, если нам нужно получить спектрограмму для тика длины BASE_TICK * W, то мы разбиваем спектрограмму на кусочки по W бейз-тиков, после чего усредняем по каждым <b>C_OVERLAP_DEGREE</b> подряд стоящим таким кусочкам.
