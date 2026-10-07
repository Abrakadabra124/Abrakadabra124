# Abrakadabra124 | DevSecOps и Platform Engineering

Развиваю платформы безопасной доставки и практические инженерные лаборатории: от Linux и автоматизации до Kubernetes, Kafka и проверяемого восстановления. Основной стек: Linux, Ansible, Python, PostgreSQL, Docker, Kubernetes, GitHub Actions и Flux. Отдельное направление в разработке - безопасность жизненного цикла машинного обучения.

Цель портфолио - показывать инженерные решения, работающие проверки и ограничения, а не только список технологий.

## С чего начать знакомство

| Проект | Что показывает | Границы результата |
|---|---|---|
| [Enterprise DevSecOps Platform](https://github.com/Abrakadabra124/enterprise-devsecops-platform) | Подписанная поставка, GitOps, security gates, мониторинг, backup и восстановление PostgreSQL | Проверенная production-like лаборатория L1 на одном host, не production HA |
| [Kafka Workbench](https://github.com/Abrakadabra124/kafka-workbench) | Локальный учебный продукт: поток событий, диагностика инцидентов, сохранение прогресса и настоящая Kafka-лаборатория | Один пользователь и один брокер, не публичный SaaS |
| [Linux Secure Baseline](https://github.com/Abrakadabra124/linux-secure-baseline) | Linux hardening, Ansible, идемпотентность, сетевая диагностика и сценарии отказов | Выпуск v1.0.0, локальная WSL2-лаборатория с общим ядром |
| [Trusted MLSecOps Platform](https://github.com/Abrakadabra124/trusted-mlsecops-platform) | Происхождение данных, изолированное обучение, подписи, независимая оценка и ролевое хранилище | Developer preview на синтетических данных; полная приёмка ещё не завершена |

Проекты выбраны по наличию реализации, воспроизводимых проверок, документации и явно описанных ограничений. Исследование, учебный roadmap и заготовка репозитория не приравниваются к готовому продукту.

<details>
<summary><strong>Enterprise DevSecOps Platform: архитектура, приёмка и измерения</strong></summary>

## Enterprise DevSecOps Platform

[Репозиторий и запуск](https://github.com/Abrakadabra124/enterprise-devsecops-platform) · [Архитектура](https://github.com/Abrakadabra124/enterprise-devsecops-platform/blob/main/docs/architecture.md) · [Инструкция по эксплуатации L1](https://github.com/Abrakadabra124/enterprise-devsecops-platform/blob/main/docs/runbooks/production-lab.md) · [GitHub Actions](https://github.com/Abrakadabra124/enterprise-devsecops-platform/actions/workflows/quality.yml)

Production-like лаборатория на одном компьютере. FastAPI предоставляет HTTP API, PostgreSQL хранит данные, Kubernetes запускает приложение. Платформа связывает доставку и эксплуатацию в один проверяемый процесс:

1. **Проверки до поставки.** CI (continuous integration, непрерывная интеграция) проверяет качество кода, тесты, работу с настоящим PostgreSQL и безопасность. Отрицательные тесты проверяют, что запрещённые сценарии действительно блокируются.
2. **Контроль артефактов и развёртывания.** Cosign подписывает артефакты, а неизменяемые digest связывают разрешённую поставку с конкретным содержимым образа. Flux приводит Kubernetes к декларативному состоянию; SOPS шифрует секреты конфигурации.
3. **Наблюдаемость.** Prometheus собирает метрики, Grafana показывает их, Alertmanager обрабатывает оповещения. Постоянные тома сохраняют состояние при замене pod; это проверяется сравнением данных до и после перезапуска.
4. **Восстановление.** Согласованный PostgreSQL backup шифруется отдельным ключом age; Cosign подписывает индекс с контрольной суммой архива. Проверка восстанавливает данные в отдельный PostgreSQL, не перезаписывая рабочую базу.
5. **Приёмка.** Скрипты проверяют конечное состояние, восстановление после сбоев, откат и защиту от подмены. Зелёный exit code отдельной команды не заменяет проверку всей цепочки.

### Проверенный локальный срез

Приёмка от **7 октября 2026 года**, исходное дерево опубликовано в [commit ee65d4a](https://github.com/Abrakadabra124/enterprise-devsecops-platform/commit/ee65d4a912d9361647312a110db1a14179201364):

| Проверка | Результат |
|---|---|
| Модульные и регрессионные тесты | 230 passed |
| Интеграционные тесты с настоящим PostgreSQL | 12 passed |
| Совокупное покрытие строк и ветвей приложения | 98,98% |
| Базовая приёмка Q01-Q10 и лабораторная L01-L04 | Все проверки passed |
| Нагрузка: 10 клиентов, около 60 секунд после прогрева | 12 079 запросов, 0 ошибок, p95 112,42 мс |
| Восстановление зашифрованного backup в отдельную БД | 14 из 14 строк, совпадение checksum, 2,64 секунды |
| Отрицательные проверки backup | Изменённый индекс, неверный ключ и повреждённый архив отклонены |

Это измерения конкретного локального прогона на небольшом наборе данных, не обещание производительности или доступности в production. GitHub Actions проверяет исходники, интеграцию и security gates; локальные Kubernetes drills не выполняются в hosted CI. Приватные отчёты, дампы, ключи и runtime-конфигурация в GitHub не публикуются. Порядок повторения проверок описан в [runbook](https://github.com/Abrakadabra124/enterprise-devsecops-platform/blob/main/docs/runbooks/production-lab.md).

### Границы и следующие этапы

- Один host остаётся общей точкой отказа. Постоянные тома защищают от замены pod, но не от потери компьютера или его диска.
- Backup пока запускается вручную и хранится на том же host. Расписание, независимое offsite-хранилище и восстановление на заданный момент ещё предстоит реализовать.
- TLS для API/registry, централизованный вход OIDC, внешнее управление ключами и независимые домены отказа пока не закрыты. Публичный production-доступ не заявляется.
- Короткий нагрузочный тест не подтверждает месячную доступность. Длительные проверки, эксплуатационные владельцы и доставка оповещений оператору остаются отдельными задачами.

</details>

## Kafka Workbench

Apache Kafka - брокер событий, позволяющий независимо записывать и читать потоки сообщений. Workbench связывает 18 уроков, шесть учебных инцидентов и визуальную модель consumer groups с настоящим брокером в Docker. React/TypeScript отвечают за интерфейс, Express - за проверку решений, SQLite - за сохранение прогресса.

Автоматические проверки проходят на Windows и Linux. Отдельное задание GitHub Actions выполняет браузерные сценарии и настоящий цикл записи/чтения Kafka с проверкой offsets. Учебная анимация не выдаётся за реальный кластер; один брокер не доказывает отказоустойчивость.

[Запуск](https://github.com/Abrakadabra124/kafka-workbench#запуск-из-github) · [Проверки и ограничения](https://github.com/Abrakadabra124/kafka-workbench/blob/main/docs/VERIFICATION.md) · [Эксплуатация](https://github.com/Abrakadabra124/kafka-workbench/blob/main/docs/OPERATIONS.md)

## Linux Secure Baseline

Ansible приводит настройки Linux к описанному состоянию; повторный запуск проверяет идемпотентность, то есть отсутствие лишних изменений. Лаборатория включает контроль SSH, sysctl, journald и limits, проверку сети и пять сценариев отказа: DNS, закрытый порт, TLS, права доступа и заполнение диска.

WSL2 (Windows Subsystem for Linux 2) позволяет запускать отдельные Linux-окружения на Windows, но их общее ядро не даёт независимых доменов отказа. Документация разделяет проверку кода в CI и локальные эксперименты с узлами.

[Релиз v1.0.0](https://github.com/Abrakadabra124/linux-secure-baseline/releases/tag/v1.0.0) · [Архитектура](https://github.com/Abrakadabra124/linux-secure-baseline/blob/main/docs/architecture.md) · [Аудит завершённого этапа](https://github.com/Abrakadabra124/linux-secure-baseline/blob/main/docs/evidence/weeks-2-4-completion-audit.md)

## Trusted MLSecOps Platform: в разработке

MLSecOps (Machine Learning Security Operations) защищает данные, процесс обучения, артефакты модели и её выпуск. В developer preview реализованы подписанное происхождение синтетических данных, обучение в изолированных Docker/Kubernetes workers, ONNX-предсказания и отдельная оценка модели. PostgreSQL с раздельными ролями и взаимной TLS-аутентификацией ограничивает доступ к хранилищу; backup восстанавливается в независимую БД.

Для текущего проверенного commit успешны четыре workflow: документация, developer runtime, Kubernetes isolation и storage/TLS. Это проверки конкретных компонентов, **не завершение всех M01-M23** и не доказательство пользы модели на реальных релизах. Целевая платформа продолжает разрабатываться.

[Статус и границы доказательств](https://github.com/Abrakadabra124/trusted-mlsecops-platform/blob/main/STATUS.md) · [Модель угроз](https://github.com/Abrakadabra124/trusted-mlsecops-platform/blob/main/docs/threat-model.md) · [Критерии приёмки](https://github.com/Abrakadabra124/trusted-mlsecops-platform/blob/main/docs/acceptance.md)

## Проверенные GitHub Actions

Снимок проверки портфолио: **8 октября 2026 года**. Ссылки относятся к конкретным commits, а не обещают успешность любого будущего изменения. CI означает continuous integration, непрерывную интеграцию; успешный workflow подтверждает только выполненные в нём проверки.

| Проект и commit | Успешные проверки |
|---|---|
| DevSecOps `ee65d4a` | [Качество, настоящая PostgreSQL-интеграция и security gates](https://github.com/Abrakadabra124/enterprise-devsecops-platform/actions/runs/37688987968) |
| Kafka `535166b` | [Windows/Linux, браузер и настоящий Kafka broker](https://github.com/Abrakadabra124/kafka-workbench/actions/runs/37658936863) |
| Linux `eb186d5` | [Shell/PowerShell, Ansible, диагностика и отрицательные сценарии](https://github.com/Abrakadabra124/linux-secure-baseline/actions/runs/33473727857) |
| MLSecOps `4486ae1` | [Docs](https://github.com/Abrakadabra124/trusted-mlsecops-platform/actions/runs/37687247827), [runtime](https://github.com/Abrakadabra124/trusted-mlsecops-platform/actions/runs/37687247959), [Kubernetes](https://github.com/Abrakadabra124/trusted-mlsecops-platform/actions/runs/37687247966), [storage/TLS](https://github.com/Abrakadabra124/trusted-mlsecops-platform/actions/runs/37687247858) |

## Roadmap и отдельные заготовки

[DevSecOps Roadmap](https://github.com/Abrakadabra124/github-middle-devsecops-roadmap) хранит исходный 24-недельный план и переход к работе от конечной цели, исследования и измеримой приёмки. Это документация развития, а не ещё одна реализованная платформа.

Отдельные учебные репозитории [Secure CI/CD](https://github.com/Abrakadabra124/secure-ci-cd), [Kubernetes Platform Lab](https://github.com/Abrakadabra124/kubernetes-platform-lab), [Terraform Infrastructure](https://github.com/Abrakadabra124/terraform-infrastructure) и [Observability & SRE Lab](https://github.com/Abrakadabra124/observability-sre-lab) не представлены здесь как завершённые продукты. Реализованные в основном проекте контуры не означают завершение этих самостоятельных репозиториев.

## Как я подхожу к работе

- Требование -> проектирование -> реализация -> защита -> проверка -> документация.
- Минимальные привилегии, секреты вне Git и явные границы доверия.
- Идемпотентные операции, фиксированные зависимости и проверка реального конечного состояния.
- Для изменений нужны сценарии отказа, откат и проверяемое восстановление.
- Проверенные результаты, планы и ограничения описываются отдельно.
