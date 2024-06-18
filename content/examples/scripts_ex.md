---
title:  Примеры скриптов
weight: 70
---

## groovy

```groovy {filename="Отчет по нормативным МПА на регистрации"}
import com.company.bratsk.entity.Mpa
import com.haulmont.thesis.core.enums.DocOfficeDocKind
import com.company.bratsk.service.CardUrlService
import com.haulmont.cuba.core.Persistence
import com.haulmont.cuba.core.Transaction
import com.haulmont.thesis.core.entity.SimpleDoc
import com.haulmont.workflow.core.entity.CardRelation
import com.haulmont.cuba.core.global.AppBeans
import com.haulmont.cuba.core.global.ViewRepository
import com.haulmont.cuba.core.global.Metadata

CardUrlService cardUrlService = AppBeans.get(CardUrlService.NAME)

def result = []; // тут храним результат, ключ-значение (ключ=ключу в шаблоне отчета)
long counter = 0

for (Mpa doc : params.entities as List<Mpa>) {
    if (doc.getState() != null && doc.getStatusNormativ() != null && doc.getState().contains('Registration')) {
        Map<String, Object> map = new HashMap<String, Object>()
        map.put("counter", ++counter)
        map.put("docKind", doc.getDocKind() != null ? doc.getDocKind().getName() : "")
        map.put("theme", cardUrlService.makeHtmlLink(doc, doc.getComment())) // наименование МПА
        map.put("prepared", doc.getPrepared() != null ? doc.getPrepared().getName() : "")
        map.put("executorUser", doc.getExecutorUser() != null ? doc.getExecutorUser().getName() : "")
        map.put("note", doc.getTheme() != null ? doc.getTheme() : "")

        // Ссылки на исходящие документы
        def cardRelations = getCardRelations(doc.getId())
        String relatedOutgoingDocsInfo = "<html><meta charset=\"utf-8\"/>" + cardRelations
                .findAll{it.relatedCard instanceof SimpleDoc && ((SimpleDoc) it.relatedCard).docOfficeDocKind.equals(DocOfficeDocKind.OUTCOME)}
                .collect{ "<a href=\"" + cardUrlService.makeLink(it.relatedCard) + "\">" + it.relatedCard.getId() + "</a>"}
                .join("<br>")
        relatedOutgoingDocsInfo = relatedOutgoingDocsInfo + "</html>"
        map.put("linksOut", relatedOutgoingDocsInfo != null ? relatedOutgoingDocsInfo : "")

        result.add(map)
    }
}
return result

// Получение связанных карточек (вкладка "Связанные документы" в карточке)
def getCardRelations(cardId){
    Persistence persistence = AppBeans.get(Persistence.NAME)
    Transaction tx = persistence.createTransaction()
    try {
        ViewRepository viewRepository = AppBeans.get(Metadata.class).getViewRepository();
        List<CardRelation> cardRelations = persistence.getEntityManager()
                .createQuery("select cr from wf\$CardRelation cr " +
                        "where (cr.card.id = :cardId " +
                        "or cr.relatedCard.id = :cardId) " +
                        "and cr.deleteTs is null " +
                        "order by cr.createTs desc"
                        , CardRelation.class)
                .setParameter("cardId", cardId)
                .setView(viewRepository.getView(CardRelation.class, "card-relation"))
                .getResultList()
        tx.commit()
        return cardRelations
    } finally {
        tx.end()
    }
}
```

```groovy {filename="Отчет по рассылке документа"}
import com.company.bratsk.entity.MailingMember
import com.haulmont.cuba.core.Persistence
import com.haulmont.cuba.core.Transaction
import com.haulmont.cuba.core.global.AppBeans
import com.haulmont.cuba.core.global.Metadata
import com.haulmont.cuba.core.global.ViewRepository
import com.haulmont.thesis.core.entity.Doc

import java.text.SimpleDateFormat

SimpleDateFormat sdf = new SimpleDateFormat("dd.MM.yyyy")

def result = []; // тут храним результат, ключ-значение (ключ=ключу в шаблоне отчета)

Doc doc = params.entity as Doc
def membersForCard = getMailingProc(doc.getId()) // Есть ли рассылка для карточки

def membersGroupedByMsg = membersForCard
        .sort { it.createTs } // для того, чтобы отображались по порядку добавления
        .groupBy { it.message }

membersGroupedByMsg.each { key, value ->
    Map<String, Object> map = new HashMap<String, Object>()
    def member = value.get(0);
    if (member != null) {
        map.put("author", member.getCreator() != null ? member.getCreator().getName() : "")
        map.put("date", member.getCreateTs() != null ? sdf.format(member.getCreateTs()) : "")
        map.put("authorDelegator", member.getSubstitutedCreator() != null ? member.getSubstitutedCreator().getName() : "")
        def adressesStr = value.collect { it.user.name }.join(", ")
        map.put("addresses", adressesStr)
    }
    map.put("text", key != null ? key : "")
    result.add(map)
}
return result

def getMailingProc(cardId) {
    Persistence persistence = AppBeans.get(Persistence.NAME)
    Transaction tx = persistence.createTransaction()
    try {
        ViewRepository viewRepository = AppBeans.get(Metadata.class).getViewRepository();
        List<MailingMember> members = persistence.getEntityManager()
                .createQuery("select mail from bratsk\$MailingMember mail " +
                        "where mail.mailingProcess.card.id = :cardId " +
                        "order by mail.createTs desc", MailingMember.class)
                .setParameter("cardId", cardId)
                .setView(viewRepository.getView(MailingMember.class, "edit"))
                .getResultList()
        tx.commit()
        return members
    } finally {
        tx.end()
    }
}
```

```groovy {filename="Выборка документов для группы доступа"}
{E}.id in (select d.card.id from bratsk$DfDocRestrictions d where
    d.user.id = :session$userId
    or (
        (d.department.id in :session$departmentIds or d.organization.id = :session$organizationId)
        and (
            d.docKind.docType.name = 'bratsk$Mpa'
            or d.docKind.docType.name = 'bratsk$OG'
            or d.docKind.docType.name = 'bratsk$MunicipalService'
            or d.docKind.code = 'outgoingDoc'
            or d.docKind.code = 'incomingDoc'
        )
    )
    or (d.global = true and d.template <> true)
    or (d.global = true and (d.organization is null or d.organization.id = :session$organizationId))
)
```

<!-- ```groovy {filename="группа доступа"}
``` -->

## sql

```SQL {filename="Все подразделения"}
select c.id, c.name, o.name from DF_DEPARTMENT AS d 
join DF_CORRESPONDENT AS c ON c.id = d.correspondent_id 
join DF_ORGANIZATION AS o ON o.id = c.organization_id
where c.delete_ts is null
```