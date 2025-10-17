---
title: Примеры UML
weight: 60
---

## Диаграммы последовательности
### Авторизация на сайте

![alt text](image.png)

{{% details title="PlantUML" closed="true" %}}
```yaml
autonumber

actor user
participant "website" as site
database "db" as db

user->site:enterCredentials()
activate user
activate site
site->db:validateLogin()
activate db
site<--db:loginResult
deactivate db
user<--site:loginResult
deactivate site
``` 
{{% /details %}}

### Процесс отправки запроса в СМЭВ
![alt text](image-3.png)

{{% details title="PlantUML" closed="true" %}}
```yaml
@startuml
title "Процесс отправки запроса в СМЭВ"

autonumber
!pragma teoz true

box "Внутренняя сеть" #LightCyan
participant "СЭД \n(443)" as sed
participant "Билдер xml-builder \n(8086)" as builder
participant "СМЭВ Адаптер \n(8085)" as adapter
end box

participant "СМЭВ" as smev

note over of smev 
**СМЭВ** - Система 
межведомственного 
эл. взаимодействия
end note 

sed -> builder++ : Запросить xml-схему для МЗ
note right: **МЗ** - Межведомственный Запрос
|||
sed <-- builder-- : xml-схема МЗ
|||
sed ->> adapter++ : Отправить МЗ (xml-схема, файлы, ЭП)
'sed <-- adapter : Сообщение об ошибке
adapter ->> smev++: Отправить МЗ
smev --> adapter --: Результат обработки МЗ
& adapter --> sed --: Результат обработки МЗ

@enduml
``` 
{{% /details %}}

### Работа с ОГ
На сайте можно подать обращение гражданина (ОГ). Эти обращения автоматически передаются в СЭД и создаются карточки входящих ОГ. При создании запускается процесс регистрации на пользователя, указанного в шаблоне для интеграции в СЭД (Шаблон для интеграции).

ОГ передаются с уникальным кодом "Уникальный идентификатор с сайта" (это поле скрыто для пользователей). На сайт обратно передается состояние в случае (см. класс TaskChangedListener):
- Принято к исполнению - если создано поручение на основании ОГ.
- Завершено - если поручение по ОГ завершено (ExtTask.isFinished(task))

![alt text](image-1.png)

{{% details title="PlantUML" closed="true" %}}
```yaml
@startuml
autonumber

title "Процесс получения обращения гражданина (ОГ) с сайта"

participant "СЭД" as sed
participant "Сайт" as site
actor "Работник по ОГ" as user

sed->>site++:GET /appeal/getAppeals
activate sed
sed<<--site--:список ОГ

loop 3. для каждого ОГ
    autonumber 3.1 
    sed->sed: saveResponse(appeal)
    note right 
        (=создать входящий ОГ)
    end note
    
    sed->user:назначить ОГ
    activate user
    ref over user 
        Процесс обработки ОГ, 
        полученного с сайта
    end ref
    site<<-sed++:GET /appeal/getSuccessOg{id}
    site-->>sed--:OK
    deactivate sed
end
@enduml
``` 
{{% /details %}}

![alt text](image-2.png)
{{% details title="PlantUML" closed="true" %}}
```yaml
@startuml
autonumber
'!pragma teoz true
'skinparam sequenceMessageAlign right
title "Процесс обработки обращения гражданина (ОГ), полученного с сайта"

participant "СЭД" as sed
participant "Сайт" as site
actor "Работник по ОГ" as user
participant "Поручение" as task

ref over sed
Процесс получения 
ОГ с сайта
end ref
sed -> user: назначить ОГ
activate sed
activate user
user->task **: создать поручение\nnew ExtTask()
activate task
task-->>sed: поручение создано
site<-sed++:PUT /appeal/sendStatus{id, status}\nstatus=Принято к исполнению
site-->sed--: OK
user->task:завершить поручение
deactivate user
task-->>sed: поручение завершено
deactivate task
site<-sed++:PUT /appeal/sendStatus{id, status}\nstatus=Завершено
site-->sed--: OK
deactivate sed

@enduml
``` 
{{% /details %}}

## Диаграммы прецендентов


## Диаграммы состояний
### Публикация МПА
![alt text](image-4.png)
{{% details title="PlantUML" closed="true" %}}
```yaml
@startuml
hide empty description

state "Опубликовать" as wait
state "Снято\nс публикации" as cancel

[*] -> wait
wait : do/ заполнить в МПА:\nназвание,издание,№
wait : exit/ клик на "Опубликовать"
wait -> Опубликование : данные\nзаполнены
Опубликование --> Опубликован : отправлено\nна публикацию
Опубликование : do/ POST /mpa/send{mpa}
Опубликование : do/ POST /mpa/sendMpaFiles{files}
Опубликован --> cancel : размещение\nнеактуально/\nошибочно
Опубликован -> [*]  : размещено\nна сайте
cancel --> [*] : удалено\nс сайта
wait <- cancel : повторить\nопубликование
cancel : entry/ клик на "Отменить публикацию" 
cancel : do/ DELETE /mpa/remove{url} 

@enduml
``` 
{{% /details %}}

### Состояния поручения
![alt text](image-5.png)
{{% details title="PlantUML" closed="true" %}}
```yaml
@startuml
hide empty description

title Состояния поручения

state "Новое" as new
state "В процессе" as inProcess
state "Отменено" as canceled
state "Завершено" as finished

[*] -> new
new --> inProcess

state inProcess {
    inProcess : entry/ клик на кнопку "Поручить" в РК поручения
    state "Назначено" as assigned
    state "В работе" as inWork
    state "Возвращено\nисполнителем" as returned
    state "Контролируемое\nпоручение" as controlling

    state isAuthor <<choice>> 

    [*] --> isAuthor
    isAuthor --> inWork : [создатель=автор]
    isAuthor --> assigned : [создатель!=автор]
    assigned --> inWork : принято\nв работу
    assigned --> controlling : выполнено\n[контролер\nназначен]
    'assigned -l-> returned : запрос переноса/\nуточнение/\nотказ
    returned <- assigned : запрос\nпереноса сроков/\nуточнение/\nотказ
    returned -> assigned : корректировка\nзавершена
    inWork --> controlling : выполнено\n[контролер\nназначен]
    'inWork --> finished : [контролер\nне назначен]

    state controlling {
        controlling : exit/ принято контролером
        state "На контроле" as onControl
        state "На доработке" as onCorrection
        onCorrection : exit/ клик на "Выполнено"

        onCorrection -l-> onControl : [замечания\nустранены]
        onControl -> onCorrection : [есть\nзамечания]
    }

    controlling --> [*] : принято\nконтролером
    inWork --> [*] : выполнено\n[контролер\nне назначен]
}

inProcess --> finished : процесс\nзавершен
inProcess --> canceled : процесс\nотменен
canceled --> [*] : процесс\nотменен
finished --> [*] : процесс\nзавершен

@enduml
``` 
{{% /details %}}

### Статусы исходящего документа
![alt text](image-6.png)
{{% details title="PlantUML" closed="true" %}}
```yaml
@startuml
hide empty description
title Статусы исходящего документа

state "Новый" as new
state "Отменен" as rejected
state "Зарегистрирован" as registered
state "Процесс запущен" as inProcess

[*] -> new

state inProcess {  
    state "Согласование ИСХ" as agreement
    state "На доработке" as onRevision
    state "На регистрации" as onRegistration

    state agreement {
        state "На утверждении" as onApproval
        state "На согласовании" as onAgreement
        state "Согласован" as agreed
        state "Утвержден" as approved

        new -> onAgreement : [согласующий\nуказан]
        new -> onApproval : [согласующий\nне указан]
        onAgreement --> onApproval : [утверждающий\nуказан]
        onAgreement --> agreed : [утверждающий\nне указан]
        onApproval --> approved
    }

    agreement --> onRevision : [есть\nзамечания]
    agreement <-- onRevision : [замечания\nустранены]
    agreed --> onRegistration
    approved --> onRegistration
    onRegistration --> registered : заполнен рег.номер, дата, дело
}

registered -> [*] : обработка завершена успешно
inProcess --> rejected : [процесс\nотменен]
rejected -> [*] : обработка\nотменена

@enduml
``` 
{{% /details %}}

## Диаграммы развертывания

{{% details title="PlantUML" closed="true" %}}
```yaml

``` 
{{% /details %}}