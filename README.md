<!--
Бланк звіту до лабораторної роботи «DNS».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Коментарі такого вигляду в GitHub не відображаються; видаляти їх не обов'язково.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота. DNS

| Поле | Значення |
|---|---|
| ПІБ | Бонарь Владислав Дмитрович |
| Група | F5 2.01 |
| Ідентифікатор (`sNN`) | s04 |
| Дата виконання | 08.11.26 |
| Середовище | GitHub Codespaces |
| Версія `dig` | DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu |

## Параметри варіанта

<!-- Переписати з картки stand/cards/sNN.md. -->

| Параметр | Значення |
|---|---|
| `$TTL`, с | 90 |
| TTL запису SOA, с | 15 |
| Поле `minimum` SOA, с | 20 |
| Псевдонім (CNAME на `www`) | docs |
| Другий хост і його TTL, с | edge (TTL: 20) |
| Пріоритет MX | 40 |
| Токен TXT | lab-token=ad25406f |
| Ім'я для завдання 5 | edge |

## Завдання 1. Складові URI

### 1.1. Розбір

<!-- Відсутню складову позначати «—». Порт за замовчуванням записувати з позначкою «(за замовч.)». -->

| № | scheme | userinfo | host | port | path | query | fragment |
|---|---|---|---|---|---|---|---|
| 1 | http | - | www.s04.lab | 80 | /search/reports | limit=95&sort=x | section-2 |
| 2 | http | - | docs.s04.lab | 80 | /reports/ | limit=%D0%BA%D0%B8%D1%97%D0%B2 | - |
| 3 | http | user:pass | edge.s04.lab | 80 | /search | - | top |
| 4 | https | - | www.s04.lab	 | 8443 | /search/../reports | - | - |
| 5 |mailto | - | - | - | admin@s04.lab | subject=search | - |

| № | Ім'я, що запитується в DNS | Заголовок `Host` | Ціль запиту (request target) |
|---|---|---|---|
| 1 | www.s04.lab | www.s04.lab | /search/reports?limit=95&sort=x |
| 2 | docs.s04.lab | docs.s04.lab | /reports/?limit=%D0%BA%D0%B8%D1%97%D0%B2 |
| 3 | edge.s04.lab | edge.s04.lab | /search |
| 4 | www.s04.lab | www.s04.lab:8443 | /search/../reports |
| 5 | - | - | - |

Декодоване значення параметра URI 2: limit=%D0%BA%D0%B8%D1%97%D0%B2 → limit=київ

Чому URI 5 не має складової host, хоча містить символ `@`: У URI mailto:admin@s04.lab?subject=search після схеми mailto: немає //, тобто немає складової authority. Тому весь фрагмент admin@s04.lab є path (адреса електронної пошти), а не userinfo@host. Символ @ розділяє userinfo і host лише всередині authority, яка починається з //. Хоча s04.lab схожий на ім'я хоста, за RFC 3986 тут він частина path, і DNS-запит для розбору URI не виконується.

### 1.2. Перевірка (після завдання 3)

<!-- Для URI 1–3: рядки запиту (починаються з ">") і тіло відповіді сервера. -->

```text
# URI 1
> GET /search/reports?limit=95&sort=x HTTP/1.1
> Host: www.s04.lab
> User-Agent: curl/8.5.0
> Accept: */*

server: web-a
Host: www.s04.lab
request URI: /search/reports?limit=95&sort=x
path: /search/reports
query: limit=95&sort=x
client: 10.53.1.1

# URI 2
> GET /reports/?limit=%D0%BA%D0%B8%D1%97%D0%B2 HTTP/1.1
> Host: docs.s04.lab
> User-Agent: curl/8.5.0
> Accept: */*

server: web-a
Host: docs.s04.lab
request URI: /reports/?limit=%D0%BA%D0%B8%D1%97%D0%B2
path: /reports/
query: limit=%D0%BA%D0%B8%D1%97%D0%B2
client: 10.53.1.1

# URI 3
> GET /search HTTP/1.1
> Host: edge.s04.lab
> Authorization: Basic dXNlcjpwYXNz
> User-Agent: curl/8.5.0
> Accept: */*

server: web-b
Host: edge.s04.lab
request URI: /search
path: /search
query: 
client: 10.53.1.1
```

Значення заголовка `Authorization` і результат декодування:

```text
Authorization: Basic dXNlcjpwYXNz
Декодовано (Base64 -> текст): user:pass
```

URI 4: етап, на якому запит не вдається (пошук імені / TCP / TLS), і рядки виводу, що це підтверджують:

```text
* Host www.s04.lab:8443 was resolved.
* IPv4: 10.53.0.80
*   Trying 10.53.0.80:8443...
* connect to 10.53.0.80 port 8443 from 10.53.1.1 port 43004 failed: Connection refused
curl: (7) Failed to connect to www.s04.lab port 8443 after 1 ms: Couldn't connect to server
```

| Запит | `request URI`, отриманий сервером | `path`, отриманий сервером |
|---|---|---|
| звичайний | `/reports` | `/reports` |
| `--path-as-is` | `/search/../reports` | `/reports` |

Розбіжності між URI та тим, що отримав сервер, і їх пояснення: Фрагменти #section-2 і #top залишаються на стороні клієнта й не передаються на сервер. Дані user:pass із URI 3 клієнт вирізає та надсилає в окремому заголовку Authorization: Basic dXNlcjpwYXNz. У URI 2 у заголовку Host передається початковий псевдонім docs.s04.lab, а параметр запиту залишається закодованим. Шлях /search/../reports клієнт за замовчуванням згортає до /reports ще до відправлення, а з прапорцем --path-as-is передає без змін у request URI, тоді як path до /reports нормалізує вже сам сервер.

## Завдання 2. Проходження ієрархії вручну

```text
dig @10.53.0.2 ns.s04.lab A +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.0.2 ns.s04.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42248
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 47e5780a3cdd2812010000006af350c00e9c88b59850a00d (good)
;; QUESTION SECTION:
;ns.s04.lab.                    IN      A

;; AUTHORITY SECTION:
lab.                    172800  IN      NS      ns1.nic.lab.

;; ADDITIONAL SECTION:
ns1.nic.lab.            172800  IN      A       10.53.0.3

;; Query time: 79 msec
;; SERVER: 10.53.0.2#53(10.53.0.2) (UDP)
;; WHEN: Tue Nov 10 17:13:36 UTC 2026
;; MSG SIZE  rcvd: 111

dig @10.53.0.3 ns.s04.lab A +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.0.3 ns.s04.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1703
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 08420c260128600c010000006af350c8c0262d4dba0b310b (good)
;; QUESTION SECTION:
;ns.s04.lab.                    IN      A

;; AUTHORITY SECTION:
s04.lab.                3600    IN      NS      ns.s04.lab.

;; ADDITIONAL SECTION:
ns.s04.lab.             3600    IN      A       10.53.1.1

;; Query time: 39 msec
;; SERVER: 10.53.0.3#53(10.53.0.3) (UDP)
;; WHEN: Tue Nov 10 17:13:44 UTC 2026
;; MSG SIZE  rcvd: 97

dig @10.53.1.1 ns.s04.lab A +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 ns.s04.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 65273
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: fbcfebef626da040010000006af350cc1cdab4a99e7916b6 (good)
;; QUESTION SECTION:
;ns.s04.lab.                    IN      A

;; ANSWER SECTION:
ns.s04.lab.             90      IN      A       10.53.1.1

;; AUTHORITY SECTION:
s04.lab.                90      IN      NS      ns.s04.lab.

;; Query time: 1 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Tue Nov 10 17:13:48 UTC 2026
;; MSG SIZE  rcvd: 97

```

| Сервер | status | flags | ANSWER | AUTHORITY | ADDITIONAL |
|---|---|---|---|---|---|
| 10.53.0.2 | NOERROR | qr | 0 | 1 | 2 |
| 10.53.0.3 | NOERROR | qr | 0 | 1 | 2 |
| 10.53.1.1 | NOERROR | qr aa | 1 | 1 | 1 |

Вивід `dig ns.sNN.lab A +trace`:

```text
dig ns.s04.lab A +trace

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> ns.s04.lab A +trace
;; global options: +cmd
.                       86122   IN      NS      a.root-servers.net.
;; Received 59 bytes from 10.53.0.53#53(10.53.0.53) in 0 ms

lab.                    172800  IN      NS      ns1.nic.lab.
;; Received 111 bytes from 10.53.0.2#53(a.root-servers.net) in 80 ms

s04.lab.                3600    IN      NS      ns.s04.lab.
;; Received 97 bytes from 10.53.0.3#53(ns1.nic.lab) in 40 ms

ns.s04.lab.             90      IN      A       10.53.1.1
s04.lab.                90      IN      NS      ns.s04.lab.
;; Received 97 bytes from 10.53.1.1#53(ns.s04.lab) in 0 ms

```

| Запит | TTL запису NS | Прапорець `aa` | Секція, у якій міститься NS |
|---|---|---|---|
| `dig @10.53.0.3 sNN.lab NS +norec` | 3600 | ні | AUTHORITY |
| `dig @10.53.1.1 sNN.lab NS +norec` | 90 | так | ANSWER |

1. Які відповіді є відсиланнями і за якими ознаками це визначено? Відсиланнями є відповіді root (10.53.0.2) і TLD (10.53.0.3). Ознаки: немає прапорця aa, секція ANSWER порожня (0), NS-записи лежать в AUTHORITY, а в ADDITIONAL міститься адреса наступного сервера. Відповідь 10.53.1.1 відсиланням не є: вона має aa, а запис розташований у ANSWER.

2. Для чого у відповіді TLD присутній запис у секції ADDITIONAL і що сталося б, якби його не було? Це glue-запис ns.s04.lab. A 10.53.1.1. Ім'я ns.s04.lab належить самій зоні s04.lab, яку делегує TLD. Без цього запису резолвер не знав би адреси сервера зони: щоб отримати адресу ns.s04.lab, треба спитати сервер зони s04.lab, а щоб його спитати, потрібна його адреса. Утворилося б замкнене коло, і резолвінг не завершився б.

3. Яка з двох копій запису NS є авторитативною і чому? Копія на сервері 10.53.1.1: вона віддається з прапорцем aa і береться з файлу зони, за яку цей сервер відповідає. Копія на TLD це лише делегування (без aa, у секції AUTHORITY), TLD за зону s04.lab не відповідає. TTL теж різні: 3600 у делегуванні та 90 з файлу зони.

## Завдання 3. Заповнення власної зони

Вивід `zone-reload`:

```text
zone s04.lab/IN: loaded serial 2026102801
OK
zone reload queued
```

<!-- Шість запитів до 10.53.1.1 з +norec, по одному на кожен рядок таблиці картки. -->

| Запис | status | Прапорець `aa` | Значення в ANSWER | TTL | Відповідає картці (так/ні) |
|---|---|---|---|---|---|
| `www` A | NOERROR | так | 10.53.0.80 | 90 | так |
| псевдонім CNAME | NOERROR | так | www.s04.lab. | 90 | так |
| `mail` A | NOERROR | так | 10.53.0.80 | 90 | так |
| `@` MX | NOERROR | так | 40 mail.s04.lab.	| 90 | так |
| `@` TXT | NOERROR | так | “lab-token=ad25406f” | 90 | так |
| другий хост A | NOERROR | так | 10.53.0.81 | 20 | так |

Порівняння відповідей резолвера та власного сервера:

```text
dig docs.s04.lab

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> docs.s04.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 63582
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;docs.s04.lab.                  IN      A

;; ANSWER SECTION:
docs.s04.lab.           90      IN      CNAME   www.s04.lab.
www.s04.lab.            90      IN      A       10.53.0.80

;; Query time: 40 msec
;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)
;; WHEN: Thu Nov 12 11:21:46 UTC 2026
;; MSG SIZE  rcvd: 75

dig @10.53.1.1 docs.s04.lab +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 docs.s04.lab +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40164
;; flags: qr aa; QUERY: 1, ANSWER: 2, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 7aa5708be08998a6010000006af5a16a62bf06eec5197b77 (good)
;; QUESTION SECTION:
;docs.s04.lab.                  IN      A

;; ANSWER SECTION:
docs.s04.lab.           90      IN      CNAME   www.s04.lab.
www.s04.lab.            90      IN      A       10.53.0.80

;; AUTHORITY SECTION:
s04.lab.                90      IN      NS      ns.s04.lab.

;; ADDITIONAL SECTION:
ns.s04.lab.             90      IN      A       10.53.1.1

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Thu Nov 12 11:22:18 UTC 2026
;; MSG SIZE  rcvd: 136

root@ns-s04:/# dig s04.lab MX

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> s04.lab MX
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 34593
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;s04.lab.                       IN      MX

;; ANSWER SECTION:
s04.lab.                90      IN      MX      40 mail.s04.lab.

;; Query time: 0 msec
;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)
;; WHEN: Thu Nov 12 11:22:35 UTC 2026
;; MSG SIZE  rcvd: 57

dig @10.53.1.1 s04.lab MX +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 s04.lab MX +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 32100
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 07a01a035c65ce51010000006af5a18e561ca73b1a016e28 (good)
;; QUESTION SECTION:
;s04.lab.                       IN      MX

;; ANSWER SECTION:
s04.lab.                90      IN      MX      40 mail.s04.lab.

;; AUTHORITY SECTION:
s04.lab.                90      IN      NS      ns.s04.lab.

;; ADDITIONAL SECTION:
mail.s04.lab.           90      IN      A       10.53.0.80
ns.s04.lab.             90      IN      A       10.53.1.1

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Thu Nov 12 11:22:54 UTC 2026
;; MSG SIZE  rcvd: 134

```

| Запит | flags | ANSWER (кількість, типи) | AUTHORITY (кількість, типи) | ADDITIONAL (кількість, типи) | TTL |
|---|---|---|---|---|---|
| псевдонім через resolver-a | qr rd ra | 2 | 0 | 1 | 90 |
| псевдонім через 10.53.1.1 | qr aa	 | 2 | 1 | 2 | 90 |
| MX через resolver-a | qr rd ra | 1 | 0 | 1 | 90 |
| MX через 10.53.1.1 | qr aa	 | 1 | 1 | 3 | 90 |

Пояснення відмінностей: Резолвер відповідає без aa, але з rd і ra: він сам шукає відповідь для клієнта і не зберігає зону. Мій сервер відповідає з aa, бо зона лежить у нього, і без ra, бо шукати за когось він не вміє. Резолвер повертає лише саму відповідь, а мій сервер додає ще NS і адреси серверів (для MX ще й адресу mail). Для docs у відповіді два записи, CNAME на www і A з адресою www, а TTL скрізь 90, бо запис щойно потрапив у кеш резолвера з повним часом.

Повідомлення `zone-check` після тимчасового додавання `@ IN CNAME www`:

```text
zone-check
dns_master_load: /lab/zone/s04.lab.zone:21: s04.lab: CNAME and other data
zone s04.lab/IN: loading from master file /lab/zone/s04.lab.zone failed: CNAME and other data
zone s04.lab/IN: not loaded due to errors.
```

1. Який TTL має кожен запис зони і звідки це значення взято? Записи www, docs, mail, @ MX і @ TXT мають TTL 90 с: це значення директиви $TTL з файлу зони, бо в самих записах TTL не вказано. Запис edge має TTL 20 с, бо його задано явно в рядку запису, і він перекриває $TTL.

2. Скільки записів містить секція ANSWER у відповіді на запит псевдоніма і чому? Два: CNAME docs → www.s04.lab. і A www.s04.lab. → 10.53.0.80. Ми питали адресу (тип A), а docs є лише псевдонімом, тому сервер віддає і сам CNAME, і адресу цільового імені www.

3. Чому запис CNAME неможливо розмістити для імені `@`? CNAME означає, що ім'я є псевдонімом іншого імені, тому для нього не допускаються інші записи. На @ вже є SOA, NS, MX і TXT, тож CNAME з ними конфліктує. Експеримент це підтвердив: zone-check видав CNAME and other data, і зона не завантажилась.

## Завдання 4. TTL і видимість змін

Два запити з інтервалом 5 с:

```text
dig www.s04.lab +noall +answer
sleep 5
dig www.s04.lab +noall +answer 
www.s04.lab.            90      IN      A       10.53.0.80
www.s04.lab.            85      IN      A       10.53.0.80
```

**Прогноз** (записати до вимірювання). Формула часу появи нової адреси через залишок R, найбільша можлива затримка: T reload + R (де R — залишок TTL запису в кеші). Найбільша можлива затримка: 90 секунд (повний TTL запису).

Перевірка, що сервер до `zone-reload` повертає стару адресу:

```text
dig @10.53.1.1 www.s04.lab +norec +short
10.53.0.80
```

Залишок R і час (крок 4), час `zone-reload` (крок 5), перевірка на сервері (крок 6):

```text
date +%T; dig www.s04.lab +noall +answer
17:04:26
www.s04.lab.            90      IN      A       10.53.0.80

zone-reload; date +%T
zone s04.lab/IN: loaded serial 2026102802
OK
zone reload queued
17:04:30

dig @10.53.1.1 www.s04.lab +norec +short
10.53.0.81
```

Очікуваний час появи нової адреси (підставити R у формулу): 17:04:30 + 90 с = 17:06:00

Вивід циклу спостереження (перші два рядки, два останні рядки зі старою адресою, рядок з новою адресою):

```text
17:04:37 10.53.0.80
17:04:39 10.53.0.80
...
17:05:53 10.53.0.80
17:05:55 10.53.0.80
17:05:57 10.53.0.81
```

Рядок `server:` з відповіді `curl`:

| Час зміни | Очікуваний час | Спостережений час | Виміряна затримка, с | Прогноз, с |
|---|---|---|---|---|
| 17:04:30 | 17:06:00 | 17:05:57 | 87 | 90 |

1. Чим пояснюється розбіжність між прогнозом і виміряним значенням? Розбіжність у 3 секунди (виміряно 87 с замість 90 с) пояснюється двома чинниками: по-перше, між фіксацією залишку $R = 90$ с (о 17:04:26) і виконанням `zone-reload` (о 17:04:30) минуло 4 секунди, тож фактичний залишок TTL у кеші на момент перезавантаження становив 86 с (кеш мав спливти о 17:05:56); по-друге, цикл спостереження опитує резолвер з кроком 2 секунди (`sleep 2`), тому зміну зафіксовано на наступній перевірці о 17:05:57 (також `dig` округлює залишок TTL до цілих секунд).

2. Якою була б найбільша можлива затримка для цього запису і за яких умов? Найбільша можлива затримка становить 90 секунд (повний `$TTL` запису в зоні). Вона виникла б, якби клієнт щойно зробив запит і резолвер закешував запис із повним TTL, а адміністратор оновив зону на авторитативному сервері в ту саму секунду.

3. Як слід діяти, щоб перенести сервіс на нову адресу, якщо TTL запису становить 86 400 с? Заздалегідь (щонайменше за 86 400 с, тобто за добу до перенесення) потрібно зменшити TTL запису до малого значення, наприклад 30–60 с, і дочекатися, доки в кешах усіх резолверів спливе старий добовий запис. Після цього змінити IP-адресу на нову (перемикання клієнтів відбудеться за 30–60 с), а після успішної міграції повернути TTL назад на 86 400 с.

## Завдання 5. Негативне кешування

**Прогноз** (записати до вимірювання). Формула за RFC 2308 і розраховане значення: min(TTL SOA, minimum SOA) = min(15 с, 20 с) = 15 секунд. Негативна відповідь (NXDOMAIN) зберігатиметься в кеші резолвера 15 секунд з моменту першого запиту.

Перевірка, що сервер до `zone-reload` повертає `NXDOMAIN`:

```text
dig @10.53.1.1 new-edge.s04.lab +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.1.1 new-edge.s04.lab +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 6051
;; flags: qr aa; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 959cfe9b677eadb9010000006af60783e7018cc7d7e98b91 (good)
;; QUESTION SECTION:
;new-edge.s04.lab.              IN      A

;; AUTHORITY SECTION:
s04.lab.                15      IN      SOA     ns.s04.lab. admin.s04.lab. 2026102806 3600 600 86400 20

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Thu Nov 12 18:37:55 UTC 2026
;; MSG SIZE  rcvd: 118
```

Запит через resolver-a (крок 3), час `zone-reload` (крок 4), перевірка на сервері (крок 5):

```text
date +%T; dig new-edge.s04.lab
18:39:31
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 17198
;; AUTHORITY SECTION:
s04.lab.                15      IN      SOA     ns.s04.lab. admin.s04.lab. 2026102806 3600 600 86400 20

zone-reload; date +%T
zone s04.lab/IN: loaded serial 2026102807
OK
zone reload queued
18:39:31

dig @10.53.1.1 new-edge.s04.lab +norec +short
10.53.0.80
```

Вивід циклу спостереження (перші два рядки, останній рядок з `NXDOMAIN`, перший рядок з `NOERROR`):

```text
18:39:31 status: NXDOMAIN
18:39:33 status: NXDOMAIN
...
18:39:43 status: NXDOMAIN
18:39:45 status: NXDOMAIN
18:39:47 status: NOERROR
```

| Негативну відповідь закешовано | Запис створено | Ім'я резолвиться через resolver-a | Виміряно, с | Прогноз, с |
|---|---|---|---|---|
| 18:39:31 | 18:39:31 | 18:39:47 | 16 | 15 |

<!-- Якщо завдання повторювалося з іншим ім'ям (new2-…), зазначити це й причину. -->

1. Чому тривалість негативного кешування визначається записом SOA, а не записом, який запитували? Тому що запитуваного запису в зоні не існує (сервер повертає `NXDOMAIN`), отже, у нього немає власного значення TTL. За стандартом RFC 2308 авторитативний DNS-сервер додає до секції `AUTHORITY` негативної відповіді запис `SOA` зони, а резолвер визначає час негативного кешування як менше зі значень TTL самого запису `SOA` та його поля `minimum`.

2. Чи збігається TTL запису SOA в секції AUTHORITY з TTL запису SOA у файлі зони? Поясніть. Так, у нашому випадку збігається і становить 15 секунд[cite: 6, 9]. За RFC 2308 авторитативний сервер підставляє у поле TTL запису `SOA` в секції `AUTHORITY` негативної відповіді значення `min(TTL SOA, SOA minimum)`. Оскільки у файлі зони `s04.lab` явний TTL запису `SOA` дорівнює 15 с, а поле `minimum` — 20 с[cite: 6, 9], меншим значенням є 15 с, що збігається з TTL запису `SOA` у файлі зони (якби TTL запису `SOA` у файлі зони був більшим за `minimum`, у секції `AUTHORITY` підставлялося б значення `minimum`).

3. Скрипт розгортання перевіряє ім'я сервісу до створення запису, а потім одразу після. Що він побачить і як це виправити? Одразу після створення запису скрипт усе одно побачить `NXDOMAIN`, оскільки перша перевірка через кешуючий резолвер закешувала негативну відповідь на час `min(TTL SOA, SOA minimum)` (15 с). Щоб цього уникнути, не можна звертатися до нового доменного імені через кешуючий резолвер до створення запису та перезавантаження зони: для перевірок під час розгортання слід опитувати безпосередньо авторитативний DNS-сервер або робити паузу, не меншу за час негативного кешування.

## Завдання 6. Затримка пошуку імені

**Прогноз** (записати до вимірювання). До яких серверів звернеться resolver-a:

| Ім'я | Перший запит | Повторний запит |
|---|---|---|
| A: нове ім'я у власній зоні | | |
| B: нове ім'я в зоні `lab.` | | |
| C: ім'я в неіснуючому TLD | | |
| D: `www.sNN.lab` після перезапуску резолвера | | |

Вивід вимірювань:

```text

```

| Ім'я | Перший запит, мс | Повторний, мс | Сервери, до яких звернувся резолвер (за журналами) |
|---|---|---|---|
| A | | | |
| B | | | |
| C | | | |
| D | | | |

Значення `time_namelookup` з `curl` після перезапуску резолвера:

Фрагменти журналів запитів root, TLD і власного сервера для рядка D:

```text

```

1. Яку затримку вносить кожен із серверів root, TLD і власний сервер? Покажіть, з яких рядків таблиці це випливає.

2. Чому в рядку A перший запит не проходить через root і TLD, хоча ім'я нове?

3. Вебсторінка завантажує ресурси з п'яти різних доменів. Як це впливає на час першого відкриття сторінки?

## Завдання 7. Інцидент: зупинка власного сервера

**Прогноз** (записати до вимірювання). Що поверне кожен резолвер після зупинки сервера та спливання TTL і скільки часу займе відповідь:

Кешування запису в обох резолверах (крок 2) і час зупинки сервера (крок 3):

```text

```

Запити із заміром часу (крок 5):

```text

```

| Запит | status або тайм-аут | ANSWER, TTL | `real` |
|---|---|---|---|
| resolver-a, перший | | | |
| resolver-a, другий | | | |
| resolver-b | | | |

Час `named-start` і вивід циклу спостереження до першої успішної відповіді (крок 6):

```text

```

Перевірка після завдання (`dig @10.53.1.1 sNN.lab SOA +norec +short`):

```text

```

### Постмортем

1. Що сталося:
2. Вплив на клієнтів resolver-a і resolver-b:
3. Хронологія з виміряним часом:
4. Чому кеш допоміг або не допоміг:
5. Що слід змінити:

1. Який TTL мав запис у відповіді resolver-b і навіщо такий TTL?

2. Порівняйте результат і тривалість двох запитів до resolver-a. Якщо вони різняться, сформулюйте гіпотезу щодо причини; якщо ні, поясніть, що робить резолвер протягом цього часу.

3. Чому після `named-start` ім'я через resolver-a зарезолвилося не одразу?

4. У якій ситуації serve-stale шкодить, а не допомагає? Пов'яжіть відповідь із завданням 4.

## Фінальний файл зони

<!-- Повний вміст stand/students/sNN/zone/sNN.lab.zone після завершення роботи. -->

```text

```

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено (RFC, документація, додатковий експеримент на стенді) і що з'ясувати не вдалося:

## Декларація використання ШІ

Використовувалися інструменти ШІ: <!-- так / ні -->

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | | | |
