Title: Настройка NTP.
Date: 2014-11-29 15:01
Authors: sbog
Slug: ntp_setup
Tags: linux, ntp, ntpd
Lang: ru

Многие люди знают, что правильное время на сервере - это хорошо, но
часто не придают этому особого значения, пока им не придется
воспользоваться каким-либо сервисом (часто распределенным), критичным к
расхождению времени между серверами. Например, kerberos по умолчанию не
примет выданный сервером билет, если время между вашей нодой и сервером
расходится более чем на 5 минут. Или, например, rabbitmq будет
разваливаться при расхождении даже в минуту. Amqp-ориентированные
приложения вообще часто очень критичны к точности времени. И тогда
возникает вопрос - "А почему оно у меня не работает, вроде все же
нормально настроено? А почему оно так долго сходится, эталонные
сервера-то доступны?" В этой статье я кратко расскажу об особенностях
NTP-протокола и об основных опциях, которые хорошо бы знать.

Для начала хочется сказать, чего здесь не будет. Если вы ищете статью по
базовой настройке ntpd - вам в другое место , статей этих тыщи. Если вы
ищете волшебную палочку, после которой время на серверах начнет
сходиться в мгновение ока - вам тоже не сюда, вам читать man ntpd,
который описывает опции -g и -q. Если же вы хотите понять некоторые
вещи, которые в ntpd реализованы так, будто it's some kind of magic - то
вы по адресу, почитайте, будет интересно. Итак, начнем.

Чтобы понимать, почему та или иная фича работает ~~неправильно~~ не так,
как вы того ожидаете, нужно понимать одну основную концепцию в NTP - то,
как NTP меняет время, если замечает, что оно расходится с эталонным. NTP
имеет два способа для изменения времени - *slew*, или плавная подгонка
и *step*, или подгонка "рывком". Рассмотрим несколько ситуаций (назовем
в них наш сервер **A**, а сервер, с которого будем синхронизировать
время, то есть "эталонный", **B**):

1\. Время A и B расходится на 30 микросекунд (0.030с).

2\. Время A и B расходится на 100 секунд.

3\. Время A и B расходится на 100 минут.

4\. Время A и B расходится на 100 минут, причем по какой-то причине время
периодически скачет.

В зависимости от настройки NTP-сервера, он будет синхронизировать время
во всех случаях по-разному. Рассмотрим поподробнее каждый случай.

1\. По умолчанию, задержка до 0.125с NTP-серверу кажется в целом
нормальной и если сервер A заметит расхождение в этих пределах, то он
будет последовательно "подгонять" локальное время к эталонному.
Происходить это будет достаточно простым способом - у счетчика
локального времени есть, назовем его так, поправочный коэффициент. Этот
коэффициент NTP-сервер и будет немного увеличивать (или уменьшать, в
зависимости от того, куда "убежало" локальное время), в результате чего
через какой-то промежуток времени время на локальном сервере и на
удаленном сравняется. Такой процесс постепенной подгонки называется
slewing.

2\. Если время A и B расходятся немного более чем на 0.125 секунд (100
секунд в нашем случае), то по умолчанию NTP сервер будет вести себя
точно так же, как в первом случае. Но хотелось бы уточнить, что
локальный коэффициент сервер не может увеличивать слишком сильно
(наверное, для того, чтобы вы не сошли с ума, смотря на часы и понимая,
что у вас секунда идет за три) и поэтому время будет подгоняться к
эталону весьма долго (несколько часов как минимум).

3\. Если время расходится на 100 минут, то NTP сервер просто не
запустится. Такое расхождение считается экстраординарным и потому NTP не
хочет решать эти проблемы и предлагает пользователю самому привести
время к норме вручную, а потом уже поддерживать его, запустив NTP сервер
еще раз. Пользователей это кардинально не устраивало и потому появилась
опция "-g", с которой и NTP сервер запускается в большинстве ОС. Эта
опция говорит нам о том, что мы хотим единоразово перепрыгнуть большое
расхождение и привести его к норме без постепенной подгонки. Такое
перепрыгивание называется steping, а промежуток, после которого сервер
не будет запускаться и этот steping нужен будет в обязательном порядке,
называется panic-gate и по умолчанию равен 1000 секунд.

4\. Мы подошли к кардинальному случаю. Представим себе, что наше время
ушло на 100 минут. Мы запускаем демон NTP с опцией "-g" чтобы
перепрыгнуть к правильному значению времени. Сервер устанавливает
правильное значение, но через какое-то время некая странная программа
(или кто-то по неосторожности) снова переустанавливает время так, что
оно сильно уходит от эталона (больше чем на значение panic-gate). Если
демон NTP заметит такой случай (а он заметит), то он напишет в лог, что
так жить нельзя, перестанет выполняться, оставит вас наедине с
проблемами и ничего синхронизировать не будет. Чтобы этого не случилось
и сервер все ж таки синхронизировал время даже в таких случаях, нужно в
конфигурационном файле (по умолчанию сегодня это /etc/ntp.conf)
прописать опцию *tinker panic 0*. Эта опция должна всегда идти первой.

Итак, наш NTP-сервер теперь стартует как нужно, время всегда
синхронизирует, что еще можно поправить? Во-первых, мы можем ускорить
сам процесс синхронизации. Чтобы это сделать, нужно понять, как этот
процесс протекает в обычных условиях. А происходит следующее: наш NTP
демон отсылает пакет на эталонный сервер, тот его анализирует и
отправляет обратно. Пакет проходит через сеть не мгновенно, а с
небольшой задержкой, которая тоже учитывается. Но протокол предполагает,
что задержка в обе стороны одинаковая, а исходя из архитектуры
IP-адресации мы можем узнать, что это совсем не всегда так. Для того,
чтобы точнее посчитать задержки в сети, демон отправляет не один пакет,
а несколько, через определенные периоды времени. Процесс опроса
эталонного сервера при этом называется polling, минимальный интервал
времени, в течение которого будут отсылаться пакеты с запросом
определяется опцией minpoll, максимальный - maxpoll. Интервалов два от
того, что NTP демон постепенно увеличивает временной промежуток опросов
для того, чтобы ~~было интереснее~~ не захламлять сеть пакетами, если
время и так уже нормально синхронизировано.

Мы можем повлиять на скорость синхронизации следующими способами:

1\. Изменить опцию minpoll (которая по умолчанию равна 6) и maxpoll
(которая по умолчанию равна 10). Цифры 6 и 10 здесь - это степени
двойки, т.е. по умолчанию минимальный интервал опросов равен 2\^6=64с, а
максимальный равен 2\^10=1024с. Стоит также добавить сюда тот факт, что
ntp для начала синхронизации локального времени должен отослать 6
пакетов. Минимальный интервал нельзя сделать меньше 3, максимальный,
соответственно, тоже нет смысла делать меньше. Это позволит значительно
увеличить частоту опроса вышестоящих серверов. Не стоит просто так
делать polling интервалы отличные от тех, что стоят по умолчанию, потому
что слишком короткие интервалы могут привести к проблемам из-за
джиттера, а слишком длинные - к проблемам расхождения времени из-за
слишком редких обновлений.

2\. Добавить опции burst и iburst. По умолчанию ntp-демон отправляет один
пакет, затем ждет время minpoll, затем отправляет еще пакет, затем опять
ждет и т.д. Таким образом минимальное время для синхронизации составит
6\*minpoll(минимально 8 секунд) = как минимум 48 секунд. Опции burst и
iburst же позволяют отправлять пакеты пачкой по несколько штук.

Опция iburst актуальна только если вышестоящий сервер недоступен. Если
это так и данная опция включена, то NTP демон вместо того, чтобы
отправлять по одному пакету каждый polling интервал, отправит 6 пакетов
с двухсекундным интервалом. Если быть точнее, демон отправит 1 пакет и
будет ждать на него ответа. Если ответ не будет получен, он дождется,
пока пройдет poll интервал и попробует снова. Если ответ будет получен
(а в случае с iburst это ответ о том, что сервер unreachable), то
оставшиеся пакеты будут отсылаться друг за другом с двухсекундной
задержкой. После того, как poll интервал закончится, операция повторится
снова.

Опция burst похожа на iburst, но актуальна только если вышестоящий
сервер доступен. Если это так и данная опция включена, то NTP демон
рассчитает количество пакетов, которое нужно отослать и будет отсылать
их по тому же принципу, что и в случае с iburst. Часто в руководствах
можно найти слова наподобие "а-та-та, никогда-никогда не ставьте опцию
burst, если сервера, с которыми вы синхронизируетесь, не ваши, ~~а не то
вас зобанят по айпи~~ потому что это создает излишнюю нагрузку на
вышестоящие сервера". Это правда лишь отчасти и не работает на коротких
minpoll и maxpoll. Происходит это от того, что количество отправляемых
пакетов в случае с опцией burst не равно всегда 6, как в случае с
iburst, а рассчитывается по простому алгоритму - количество отправляемых
пакетов равно (\<текущее значение poll (мы же помним, что демон его
постепенно увеличивает)\> - \<minpoll\>) \^ 2. Т.е. если у нас текущий
poll интервал равен 6, минимальный также равен 6, то отправится
(6-6)\^2=1 пакет. Соответственно, если держать разницу между minpoll и
maxpoll в коротком диапазоне, ничего дурного не случится.

3\. Добавить опцию stepout. Это имеет смысл только если у вас прописан
tinker panic 0 и сейчас я расскажу, почему. Как я уже говорил выше,
tinker panic 0 выключает параноидальный режим ntp и позволяет вам иметь
синхронизированное время даже в том случае, если оно почему-то
периодически скачет в разные стороны. Но даже в таком случае NTP демон
не будет делать stepping времени быстро. Произойдет следующее -
локальный NTP демон отправит пакет на вышестоящий сервер и в ответе на
этот пакет узнает, что время расходится на непозволительно большое
значение. После этого NTP демон подумает некоторое время (по умолчанию
это 300 секунд в современных реализациях ntpd), в течение которого будет
работать как ни бывало по обычному графику - делать polling, slew-ить
время и т.п., а по истечению этого времени еще раз сравнит текущее время
с временем сервера. Если оно снова окажется непозволительно большим
(напомню, что в текущей реализации это все, что больше 0.125с), то в
этом случае уже будет сделан step. Значение stepout можно уменьшить с
300 секунд до более удобного значения.

Следует понимать, что если вы закрутите все интервалы на минимум, то
чуда все равно не произойдет - ntp не будет работать мгновенно, потому
что есть еще jitter, который, фактически, нигде не настраивается и из-за
него время будет синхронизироваться достаточно долго (при первом запуске
ntpd около 5 секунд, если время внезапно и сильно убежит в процессе
работы - около 2.5 минут, лучших результатов добиться сложно). Но, что
хуже - такое закручивание гаек приведет к ухудшению точности локального
времени (хотя если для вас точность в пределах 0.5с не играет большой
роли, то этим можно пренебречь). Ничего тут не поделать, так устроен сам
протокол.

~~Во вторых строках своего письма~~ Теперь поговорим о второй части
настройки ntpd, в которой часто происходит путаница - опции restrict.
Данная опция настраивает разнообразные ограничения, с которыми будут
сравниваться пакеты, приходящие на наш NTP сервер и на их основе будут
приниматься решения, отвечать ли на эти пакеты. Чтобы понимать, как
работают ограничения в ntpd, нужно знать две вещи:

1\. Как работают ограничения. Вы можете указать несколько опций restrict
в ntp.conf. В каждой из них необходимо указать IP-адрес, с которым будет
сравниваться исходящий адрес пришедшего к нам пакета (или же мы можем
указать IP-подсеть и маску для нее). Строки с опцией restrict будут
интерпретироваться последовательно и пришедший пакет будет сравнен с
*каждой* из этих строк. После этого сравнения ограничения из строки с
самым точным попаданием будет применены к пакету.

2\. Что значат опции, идущие после restrict. Я не буду приводить все
опции, а расскажу лишь о тех, в которых обычно путаются. Это следующие:

default - это просто замена выражения 0.0.0.0 mask 0.0.0.0, т.е. если
после restrict написать default, то все пакеты попадут под такое
ограничение (и к тем из них, которые не попадут под одну из более узких
масок далее, будут применены ограничения из этой строки)

noquery - эта опция НЕ ЗАПРЕЩАЕТ запросы времени. Она запрещает только
специальные пакеты, которые запрашивают состояние нашего сервера.

nomodify - это аналог noquery, который запрещает изменять состояние
нашего сервера удаленно.

noserve - запрещает все, кроме query и modify

ignore - запрещает вообще все. Стоит отметить, что под ограничения
попадают буквально все пакеты, приходящие к демону, в том числе и ответы
вышестоящих серверов. Если вы установите ограничение ignore в resrtict-е
default и не отмените это ограничение каким-нибудь образом ниже - то ваш
сервер не будет делать ничего вообще.

Вот, в общем-то и все сложности, с которыми сталкиваются новички при
настройке ntpd. Если вас интересует эта тема - пишите мне, я с
удовольстием расскажу подробности той или иной опции ntpd и попробую
описать почему было выбрано то или иное решение в реализации.
