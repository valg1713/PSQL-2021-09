Предисловие.  
Пишу с задержкой, но лекцию, раскрывающую таинство войны с pg_upgradeclusters не смотрел.  
К машинам подключаюсь через Putty, но уже с помощью gcloud compute ssh  %instance_name%

# Cоздание и настройка ВМ.  
проект называется postgres2021-23061980  
виртуалку создавал через GUI, добавил разрешения на почту ifti@yandex.ru.  
Сделал виртуалку через веб-морду  
Сделал диск через веб-морду  

Разметил его  
`sudo parted /dev/sdb mklabel gpt && sudo parted -a opt /dev/sdb mkpart primary ext4 0% 100% && sudo mkdir /mnt/disk1 && sudo mount /dev/sdb1 /mnt/disk1 && sudo chown -R postgres:postgres /mnt/disk1
sudo nano /etc/fstab --чтобы монтировался при запуске`

Поставил PostgreSQL 13, наивно полагая, что апгрейд кластера пройдет успешно.  

Остановил службу postgresql. Cловил варнинг, что для правильного отображения статуса статуса кластера надо пользоваться конструктом `sudo systemctl stop postgresql@13-main`, иначе systemd-unit будет в статусе failure.
Поэтому дальше буду пользоваться им.
`$sudo systemctl stop postgresql@13-main`

перенес содержимое /var/lib/postgresql/ в /mnt/disk1/13
исправил data_derectory на /mnt/disk1/13/main

Запуск!  
`$sudo systemctl start postgresql@13-main`  
проверка запуска  

`$pg_lsclusters`  
Ver Cluster Port Status Owner    Data directory     Log file  
13  main    5432 online postgres /mnt/disk1/13/main /var/log/postgresql/postgresql-13-main.log

Ставим 14 PSQL для танцев с бубнами с pg_upgradecluster  
`$sudo apt-get install postgresql-14 && sudo pg_lsclusters`

Ver Cluster Port Status Owner    Data directory              Log file  
13  main    5432 down   postgres /mnt/disk1/13/main          /var/log/postgresql/postgresql-13-main.log  
14  main    5433 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log  
  
И грохаем создавшийся автоматом кластер 14 PSQL  
`$ sudo pg_dropcluster 14 main && sudo pg_lsclusters`  
Ver Cluster Port Status Owner    Data directory     Log file  
13  main    5432 down   postgres /mnt/disk1/13/main /var/log/postgresql/postgresql-13-main.log  
  
Пробуем обновить кластер  
`$ sudo pg_upgradecluster 13 main /mnt/disk1/14/ --можно было еще -v 14 поставить, ноу нас всего две версии.`  

Кластер апгрейдиться не хочет, внятного ничго не говорит, только в логах виден fast shutdown.  
Впадаем в панику. Спустя 4 итерации с виртуалками - успокаиваемся.  
Долго пытаемся понять, что, собственно происходит. Кластер после выпонения команды выше не апгрейдится и не стартует.  
Выполнил следующее:  
`$ ps -aux |grep postgresql`
postgres   32991  0.0  0.7 220768 30304 ?        Ss   03:44   0:00 /usr/lib/postgresql/13/bin/postgres -D /mnt/disk1/13/main -c config_file=/etc/postgresql/13/main/postgresql.conf  
`$ kill 32991`  
`$ sudo pg_upgradecluster 13 main /mnt/disk1/14/`  
`****оверквотинг почикан****`  
   Success. Please check that the upgraded cluster works. If it does,you can remove the old cluster with  
   pg_dropcluster 13 main
   Ver Cluster Port Status Owner    Data directory     Log file  
   13  main    5433 down   postgres /mnt/disk1/13/main /var/log/postgresql/postgresql-13-main.log  
   Ver Cluster Port Status Owner    Data directory Log file  
   14  main    5432 down   postgres /mnt/disk1/14/ /var/log/postgresql/postgresql-14-main.log
Затаив дыхание, жму  
`$ sudo pg_ctlcluster 14 main start && pg_lsclusters`  
`Ver Cluster Port Status Owner    Data directory     Log file`  
`13  main    5433 down   postgres /mnt/disk1/13/main /var/log/postgresql/postgresql-13-main.log`  
`14  main    5432 online postgres /mnt/disk1/14/     /var/log/postgresql/postgresql-14-main.log`  

Ура, у нас есть кластер 14й версии!  
Проверим, что там внутри  
`$ sudo -u postgres psql  
app-# \l` 
                                List of databases  
     Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges  
  -----------+----------+----------+---------+---------+-----------------------  
  app       | postgres | UTF8     | C.UTF-8 | C.UTF-8 |  
  postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |  
  template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +  
            |          |          |         |         | postgres=CTc/postgres  
  template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +  
            |          |          |         |         | postgres=CTc/postgres  
  (4 rows)  
  app-# \q  

Ффух, вроде все на месте.
Теперь убиваем старый кластер.  
`$ sudo pg_dropcluster 13 main && pg_lsclusters`  
Ver Cluster Port Status Owner    Data directory Log file  
14  main    5432 online postgres /mnt/disk1/14/ /var/log/postgresql/postgresql-14-main.log

`$ sudo -u postgres psql  
app-#\c app
app-#create table test(c1 text);
app-#insert into test values('1');\q
`
Данные добавлены
Вот. У нас есть кластер, который физически лежит на внешнем диске (без всяких сим-линков), который был успешно обновлен с 13 до 14 версии.  

Задание со *: вприниципе - я так начал делать с самого начала. Это зачтется? Ошибки, которые были, устранил, руки выпрямил. Вроде.
Из сложностей - возникла непонятка при создании симлинка с /mnt/disk1/13 на /var/lib/postgresql/13. Пару раз ухитрялся переместить базу вникуда и пролюбить сами файлы. Списываю на болезнь и общую невнимательность.  
Вотрая непонятка - можно ли впринципе пользоваться симлинками в таком случае? Или это приведет к повышенному износу ресурса и/или снижению скорости работы. Вам не попадалась подобная статистика?
PS Markdown - зло.





