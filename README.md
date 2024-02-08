<img src="images/logo/logo.png" width=10%>

**Natch v.3.0**

[Телеграм-канал поддержки Natch](https://t.me/ispras_natch)
____
_В связи с переходом на новую версию аппаратных ключей инструмента лицензирования Sentinel до окончания действия всех выданных лицензий будут поддерживаться два варианта дистрибутива. Если у вас старые ключи Sentinel, то следует брать дистрибутив из папки Sentinel, если вы новый пользователь *Natch* - дистрибутив для вас в папке Sentinel_new. Так же рекомендуется переустановить окружение (aksusbd_*current_version*\_amd64.deb для Ubuntu/Debian/Astra, haspd-_*current_version*\_.x86_64.rpm для Alt), пакет находится в папке с дистрибутивом._
____

Natch (Network Application Tainting Can Help) - это инструмент для определения поверхности атаки, основанный на полносистемном эмуляторе Qemu.

Основная функция Natch - получение списка модулей (исполняемых файлов и динамических библиотек) и функций, используемых системой во время выполнения задачи.

Результат работы инструмента представлен множеством интерактивных аналитик, которые собраны в веб-интерфейсе *SNatch*.







[1. Что такое Natch](1_natch.md)

[2. Установка и настройка Natch](2_setup.md)

[3. Пошаговое руководство по работе с Natch](3_quickstart.md)

[4. Настройка окружения для работы с Natch](4_setup_env.md)

[5. Создание проекта](5_create_project.md)

[6. Определение источников пометки](6_taint_source.md)

[7. Запись и воспроизведение сценариев](7_scenario_work.md)

[8. Анализ поверхности атаки с помощью SNatch](8_snatch.md)

[9. Дополнительные возможности Natch](9_additional.md)

[10. Автоматизация процессов](10_automation.md)

[11. Полезные утилиты и скрипты Natch](11_utils.md)

[12. Примеры использования Natch](12_applications.md)

[13. FAQ](13_faq.md)

[Приложение 1. Настройка окружения для использования лицензированного Natch](14_app_license.md)

[Приложение 2. Командная строка эмулятора QEMU](15_app_qemu_cmdline.md)

[Приложение 3. Конфигурационные файлы Natch](16_app_configs.md)

[Приложение 4. Формат списка исполняемых модулей](17_app_module_cfg.md)

[Приложение 5. Формат файла с покрытием кода](18_app_coverage.md)

[Приложение 6. Команды монитора QEMU для работы с Natch](19_app_natch_cmds.md)

[Приложение 7. Требования и ограничения Natch](20_app_requirements.md)

[Приложение 8. Рекомендации по подготовке объекта оценки](21_app_oo_preparation.md)

[Приложение 9. История релизов Natch](22_app_releases.md)

-----

[Видеозаписи вебинаров](https://nextcloud.ispras.ru/index.php/s/natch_webinars)

[Практическое применение Natch](trophies.md)
