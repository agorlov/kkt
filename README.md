Http-сервис для пробития чеков.

В соответствии с 54-ФЗ продавец должен отправить электронный чек покупателю. В рабочее время это делает кассир,
но как быть если оплата произведена вне рабочего времени? Например покупка на сайте. Для этого я разработал данный
HTTP-сервис для взаимодействия сайта и ККТ в автоматическом режиме.

На сервис передается строка данных JSON POST-запросом, в которой содержатся данные чека.
Вызывается wsgi-скрипт, который взаимодействуя с дравером ТО пробивает чек на ККТ
и чек отправляется покуптелю. Смена открывается и закрывается автоматически.ККТ подключен по ethernet-интерфейсу. 
Параметры подключения задаются в файле conf.py. Модель ККТ: Атол 55Ф.

Для работы сервиса необходимо:
1.  Ubuntu + Apache web-сервер с установленным wsgi-mod.
2.  Драйвер АТОЛ ККТ 9.x (я работал с 9.10.1.5756)
        скачать можно тут http://fs.atol.ru -> Файловый архив -> Программное обеспечение - ДТО

Для установки wsgi-mod на web-сервере выполните:
    sudo apt install libapache2-mod-wsgi

Для проверки что mod-wsgi подгрузился выполните:
    sudo apache2ctl -M
выведется список загруженных модулей, в котором должа присутствовть строка wsgi_module (shared)

АТОЛ драйвер ККТ распакуйте и скопируйте файлы из папки с подходящей архитектурой в папку где лежит wsgi-скрипт
Файлы dto9base.py и dto9fptr.py из папки python в папку где лежит wsgi-скрипт
    В данном примере в папку /var/www/kkt

В файле conf.py пропишите параметры подключения к ККТ
    IP адрес, порт, путь к лог файлу и т.д.
    Параметр "Model" сотрим в Руководстве программиста приложение 7 модели ККМ
    Параметр "Test mode" - признак тестового режима. Если True, то метод на ККМ выполнен не будет (не будет ничего
    напечатано на чеке), но ее успешное выполнение (ResultCode = 0) сигнализирует о том,
    что при данном состоянии ККМ метод может быть выполнен без ошибок.

Создаем виртуальный хост apache /etc/apache2/sites-enabled/kkt

    sudo nano /etc/apache2/sites-available/000-default.conf
    Добавляем настройку:

    <VirtualHost *:80>

        ServerName 192.168.x.x
        ServerAlias localhost
        DocumentRoot "/var/www/kkt"
        <Directory /var/www/kkt>
        AddDefaultCharset utf-8
        Order allow,deny
        Allow from all
        </Directory>

        WSGIScriptAlias /kkt /var/www/kkt/kkt.wsgi

        LogLevel info

    </VirtualHost>

    WSGIPythonPath /var/www/kkt

Выполняем команду
    sudo service  apache2 restart

Заходим на страницу http://http://192.168.x.x/kkt. Должны увидеть результат выполнение команды GetStatus():
        
        Статус ККТ:
        (0, u'Ошибок нет', 0, u'Ошибок в параметрах нет')
        
Если возникли ошибки, смотрим /var/log/apache2/error.log и лог файл, путь к котрому указан в conf.py

Формирование POST-запроса к сервису.
На сервис нужно отправить JSON данные чека.

        {"DocNumber": "ТР00-003655", # Номер документа
        "DocDate": "06.10.2017",    # Дата документа
        "DocSumm": 1950,            # Сумма документа
        "Goods": {                  # Товары
                "Position_1": {     # Позиция товара
                        "Name": "Наименование товара",  # Наименование товара
                        "Price": 325.12,                # Цена товара
                        "Quantity": 6,                  # Количество товара
                        "Tax": 18,                      # НДС
                        "PositionSumm": 1950            # Сумма по позиции
                        }
                }
        }

Сервис напечатает чек на ККТ и вернет JSON ответ, в котором будет номер чека, код результата и описание результата.

Я вызываю сервис из 1С таким способом:

        // Функция формирует POST-запрос для HTTP сервиса.
        // Возвращает ответ с номером пробитого на ККТ чека.
        Функция ПробитьЧекНаККТ(ДокументОплаты) Экспорт

            Результат = 0;

            Если ДокументОплаты.НомерЧекаККМ <> 0 Тогда 
                Возврат Результат;
            КонецЕсли;

            ПараметрыФискализацииЧека = ДенежныеСредстваВызовСервера.ПараметрыЧека(ДокументОплаты, "");
            СтруктураДанных = Новый Структура;
            СтруктураДанных.Вставить("DocNumber", ДокументОплаты.Номер);
            СтруктураДанных.Вставить("DocDate", Формат(ДокументОплаты.Дата, "ДФ=dd.MM.yyyy"));
            СтруктураДанных.Вставить("DocSumm", ДокументОплаты.СуммаДокумента);
            СтруктураТовары = Новый Структура;

            НомерСтрокиТовара = 0;
            Для Каждого СтрокаМассива Из ПараметрыФискализацииЧека.ПозицииЧека Цикл
                НомерСтрокиТовара = НомерСтрокиТовара + 1;
                СтруктураСтрокаТовара = Новый Структура;
                СтруктураСтрокаТовара.Вставить("Name", СтрокаМассива.Наименование);
                СтруктураСтрокаТовара.Вставить("Price", СтрокаМассива.Цена);
                СтруктураСтрокаТовара.Вставить("Quantity", СтрокаМассива.Количество);
                СтруктураСтрокаТовара.Вставить("Tax", СтрокаМассива.СтавкаНДС);
                СтруктураСтрокаТовара.Вставить("PositionSumm", СтрокаМассива.Сумма);
                СтруктураТовары.Вставить("Position_" + НомерСтрокиТовара, СтруктураСтрокаТовара);
            КонецЦикла;

            СтруктураДанных.Вставить("Goods", СтруктураТовары);

            ЗаписьJSON = Новый ЗаписьJSON;
            ЗаписьJSON.УстановитьСтроку(Новый ПараметрыЗаписиJSON(,Символы.Таб));
            НастройкиСериализации = Новый НастройкиСериализацииJSON();
            НастройкиСериализации.СериализовыватьМассивыКакОбъекты = Ложь;
            ЗаписатьJSON(ЗаписьJSON, СтруктураДанных, НастройкиСериализации);
            СтрокаДанных = ЗаписьJSON.Закрыть();        

            // Выполнение запроса HTTP к сервису.
            Попытка
                АдресСервера = СокрЛП(Константы.IPАдресHTTPСервисаККТ.Получить());
                Соединение = Новый HTTPСоединение(АдресСервера);
            Исключение
                ТекстОшибки = нСтр("ru='Отсутствует соединение с сервером'");
                ЗаписьЖурналаРегистрации("ККТ", УровеньЖурналаРегистрации.Ошибка,,, ТекстОшибки + Символы.ПС
                                        +ИнформацияОбОшибке());
                Возврат Результат;
            КонецПопытки;

            ИмяСервиса = Константы.Получить();
            Если Лев(ИмяСервиса, 1) <> "/" Тогда
                ИмяСервиса = "/" + ИмяСервиса;
            КонецЕсли;
            HTTPЗапрос = Новый HTTPЗапрос(ИмяСервиса);
            HTTPЗапрос.Заголовки.Вставить("Content-Type", "application/x-www-form-urlencoded;charset=utf-8");
            HTTPЗапрос.УстановитьТелоИзСтроки(СтрокаДанных, КодировкаТекста.UTF8,
                                                ИспользованиеByteOrderMark.НеИспользовать);
            HTTPОтвет = Соединение.ОтправитьДляОбработки(HTTPЗапрос);
            КодСостояния = HTTPОтвет.КодСостояния;
            ТелоОтвета = HTTPОтвет.ПолучитьТелоКакСтроку();
            Если ЗначениеЗаполнено(ТелоОтвета) Тогда
                ЧтениеJSON = Новый ЧтениеJSON;
                Попытка
                    ЧтениеJSON.УстановитьСтроку(ТелоОтвета);
                    Результат = ПрочитатьJSON(ЧтениеJSON);
                    ЧтениеJSON.Закрыть();
                Исключение
                    ЗаписьЖурналаРегистрации("ККТ", УровеньЖурналаРегистрации.Ошибка,,, "Ответ сервера: " + ТелоОтвета +
                                                Символы.ПС + "Ответ ожидается в JSON формате!");
                    Возврат Результат;
                КонецПопытки;

                Если Результат.result_code <> 0 Тогда 
                    ЗаписьЖурналаРегистрации("ККТ", УровеньЖурналаРегистрации.Ошибка,,, "Не пробит чек на оплату с сайта!" +
                                                Символы.ПС + Строка(ДокументОплаты));
                Иначе 
                    ЗаписьЖурналаРегистрации("ККТ", УровеньЖурналаРегистрации.Информация,,, "Пробит чек. Номер чека: " +
                                                Результат.check_number + Символы.ПС + "Код ответа: " + 
                                                Результат.result_code + Символы.ПС + "Описание ответа: " +
                                                Результат.result_description);
                    Возврат Число(Результат.check_number);
                КонецЕсли;
            Иначе 
                ЗаписьЖурналаРегистрации("ККТ", УровеньЖурналаРегистрации.Ошибка,,, "Нет ответа от HTTP-сервиса");
            КонецЕсли;

        КонецФункции
        
Приветствуются любые замечания и советы.
