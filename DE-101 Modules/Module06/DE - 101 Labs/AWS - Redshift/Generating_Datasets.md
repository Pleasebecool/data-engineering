# Создание выборки данных

## Предварительная настройка

Нам необходимо создать EC2 инстанс с диском размером в 8Gb. В идеале нам лучше использовать t2.micro, можна использовать менший инстанс (генерация данных тогда займет больше времени). И не забудьте прикрепить IAM роль.

## Руководство по решению

1. Итак, давайте подключимся к созданному вами экземпляру EC2
    
    ``ssh -i datalearn.pem ec2-user@ec2-3-19-26-141.us-east-1.compute.amazonaws.com``

2. Мы будем пользоваться библиотекой для генерации данных [TPC-H benchmark kit](httpsgithub.comgregrahntpch-kit). Сначала нам необходимо предустановить необходимие библиотеки для установки генератора.
    
    ``sudo yum install make git gcc``

3. Клонируем репозиторий и переходим в склонированую директорию
    
    ``git clone https://github.comgregrahn/tpch-kit``
    ``cd tpch-kit/dbgen``

4. Забускаем билд библиотеки для операционой системы нашего инстанса
    
    ``make OS=LINUX``

5. Создаем дерикторию для нашых данных 
    
    ``cd $HOME``
    ``mkdir emrdata``

6. Пропипеременую окружение для библиотеки генерации данных
    
    ``export DSS_PATH=$HOME/emrdata``

7. Запускаем генерацию  данных
    
    ``cd tpch-kit/dbgen``
    ``./dbgen -v -T o -s 1``

    -v - для подробного режимаbr
    -T - для уточнения наших таблиц
    o - для 2 таблиц, которые мы создадимbr
    -s - для размера данных 1Gb

8. Переходим в дирикторию с сгенерироваными данными
    
    ``cd $HOME/emrdata``
    ``ls``
   
    увидим два созанных файла ``lineitem.tbl orders.tbl``

9. Сейчас в s3 создадим bucket чтоб подгрузить туда сгенерированые данные

    ``aws s3api create-bucket --bucket bigdatalabs --region us-east-1``
    - имя bucket должно быть уникальным
    - используй такойже регион как и для ec2 инстанса что-б уменшить разходы на сервисы в AWS

10. копируем созданные файлы в бакет

    ``aws s3 cp $HOME/emrdata s3://bigdatalabs/emrdata --recursive``
    
11. Давай поделим ети файлы на более мелкие, ето поможет нам подгрузить данные в s3 хранилище быстрее и является хорошей практикой для работы с redshift. Сначало проверим количество строчек в файле orders.tbl

   ``wc -l orders.tbl``
     1500000 orders.tbl

12. Давайте разобьем файл на 3 части
    
    ``split -d -l 500000 -a 3 orders.tbl orders.tbl.``

13. так же разобьем файл lineitem.tbl

    ``wc -l lineitem.tbl``
    ``6000000 lineitem.tbl``
    ``split -d -l 2000000 -a 3 lineitem.tbl lineitem.tbl.``

14. На выходе получим

    lineitem.tbl
    lineitem.tbl.000
    lineitem.tbl.001
    lineitem.tbl.002
    lineitem.tbl.003
    orders.tbl
    orders.tbl.000
    orders.tbl.001
    orders.tbl.002

15. Нам уже не нужны файлы lineitem.tbl и orders.tbl

    ``rm lineitem.tbl orders.tbl``

16. Копируем новые данные в s3 хранилище

    ``aws s3 cp $HOME/emrdata s3://bigdatalabs/redshiftdata --recursive``

17. Инстанс EC2 нам уже не нужен, можем его отключить
