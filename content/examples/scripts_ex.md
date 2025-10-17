---
title:  Примеры скриптов
weight: 70
---

## SQL
### SELECT

```SQL {filename="Все подразделения"}
select c.id, c.name, o.name from DF_DEPARTMENT AS d 
join DF_CORRESPONDENT AS c ON c.id = d.correspondent_id 
join DF_ORGANIZATION AS o ON o.id = c.organization_id
where c.delete_ts is null
```

```SQL {filename="Вывод организаций, РК которых не прошли миграцию"}
select distinct intt.int_corr_org as "id ОШС", dd.gid, dd.name as "Наименование"
, dd.unit_type as "Подразделение?", dd.org_gid, dd.org_name, dd.enabled
, cor.hcm_gid
, mig.message as "Ошибка миграции" --, mig.data_id as "Идентификатор РК"
from dmmigration_cat_records_queue mig
left join dmnodes_d_incoming_common__int_corr intt on mig.data_id = intt.system_id
left join dmhcm__org_and_deps dd ON dd.ID = intt.int_corr_org
left join dmrefs__corr_attr_view cor on intt.int_corr_org = cor.HCM_GID
where mig.status = '-1'
and mig.CONTROLLER_KEY like 'incomings_migration_controller'
and unit_type=0 -- чтобы вывести только организации
order by dd.name
```

```SQL {filename="Вывод организаций, РК которых не прошли миграцию"}
select distinct intt.int_corr_org as "id ОШС", dd.gid, dd.name as "Наименование"
, dd.unit_type as "Подразделение?", dd.org_gid, dd.org_name, dd.enabled
, cor.hcm_gid
, mig.message as "Ошибка миграции" --, mig.data_id as "Идентификатор РК"
from dmmigration_cat_records_queue mig
left join dmnodes_d_incoming_common__int_corr intt on mig.data_id = intt.system_id
left join dmhcm__org_and_deps dd ON dd.ID = intt.int_corr_org
left join dmrefs__corr_attr_view cor on intt.int_corr_org = cor.HCM_GID
where mig.status = '-1'
and mig.CONTROLLER_KEY like 'incomings_migration_controller'
and unit_type=0 -- чтобы вывести только организации
order by dd.name
```

### UPDATE
```SQL {filename="Корректировка шаблонов под новые атрибуты в РК"}
BEGIN
    update DMREFS__NOTIFICATIONS_TEMPLATES 
    set SUBJECT_TEMPLATE = replace(SUBJECT_TEMPLATE, '{ext_corr_name}{int_corr_name@lastname_fm}', '{correspondent_fio}'),
    body_template = replace(body_template, '{ext_corr_name}{int_corr_name@lastname_fm}', '{correspondent_fio}')
    where code = 'wf_TaskAssigned' or code like 'wf_TaskAssigned@incoming' or code like 'wf_notify_proxy@incoming';

    commit;
END;
```

```SQL {filename="Отключить автозагрузку данных в справочниках"}
DECLARE
	cnt number;
BEGIN
    SELECT COUNT(*) into cnt FROM dmrefs_refs_list WHERE autoload = 1 and nickname in ('attorney_journal', 'esign_proxies_deadlines', 'esign_cert_data', 'xde_nodes_boxes', 'corr_attr', 'lc_settings', 'proxy_log');
	if cnt > 0 THEN
        update dmrefs_refs_list set autoload = 0
        where nickname in ('attorney_journal', 'esign_proxies_deadlines', 'esign_cert_data', 
        'xde_nodes_boxes', 'corr_attr', 'lc_settings', 'proxy_log');
        commit;
	END IF;
END;
```

### INSERT INTO

```SQL {filename="Добавить атрибут в трансформацию ИСХ->ВХ"}
DECLARE
	cnt number;
BEGIN
    SELECT COUNT(*) into cnt from dmnodes_transform_mapping where transform_type like 'IncFromOutReg' and attr_function like 'getInitiatorRCCWithCondition';
	if cnt = 0 THEN
		insert into dmnodes_transform_mapping (ID, TRANSFORM_TYPE, MAPPING_TYPE, SCR_ATTR, DST_ATTR, CONSTANT_VALUE, ATTR_FUNCTION) 
        VALUES(SEQ_DMNODES_TRANSFORM_MAPPING.NEXTVAL, 'IncFromOutReg', 'attr_function', 'category@common.initiator_rcc', 'category_common@incoming.out_initiator_rcc', NULL, 'getInitiatorRCCWithCondition'); 

        commit;
	END IF;
END;
```

```SQL {filename="Добавить статус Возвращено со сканирования"}
declare
    cnt int:= 0;
    max_id int:= 0;
	p_id number;
	query varchar2(2048);
begin
	select count(id) into cnt from dmrefs__states where code = 'returned_from_scan';
	if cnt = 0 then
		p_id := 4168;
		insert into dmrefs__states (id, code, name, doc_type, workflow, enabled) values(p_id, 'returned_from_scan', 'Возвращено со сканирования', null, 0, 1);
		insert into dmrefs__states_doc_type (link_id, doc_type) values(p_id, 3);
		insert into dmrefs__states_doc_type (link_id, doc_type) values(p_id, 7);
		insert into dmrefs__states_doc_type (link_id, doc_type) values(p_id, 4);
		insert into dmrefs__states_doc_type (link_id, doc_type) values(p_id, 330);
		insert into dmrefs__states_ml (link_id, locale, name) values(p_id, 'en_US', 'Returned from scanning');
		insert into dmrefs__states_ml (link_id, locale, name) values(p_id, 'ru_RU', 'Возвращено со сканирования');

		select max(id) into max_id from dmrefs__states;
    	query := N'ALTER SEQUENCE SEQ_DMREFS__DOC_KINDS restart start with ' || max_id;
		EXECUTE IMMEDIATE query;

	    dbms_output.put_line('Статус "Возвращено со сканирования" создан' );
	ELSE 
		dbms_output.put_line('Статус "Возвращено со сканирования" УЖЕ существует' );
	end if;

    commit;
end;
```

### CREATE OR REPLACE VIEW
```SQL {filename="Корректировка view для нового атрибута в РК"}
CREATE OR REPLACE VIEW DMREFS__CORR_CURRENT_VIEW AS
select gid as id, partner_type, name, idx, country, city, street, building, is_system, acts, enabled
	from (
	select cor.*, 
		nvl(names.name, nvl(names.last_name, ' ') || ' ' || nvl(names.first_name, ' ') || ' ' || nvl(names.middle_name, ' ')) as name,
		addresses.idx, addresses.country, addresses.city, addresses.street, addresses.building,
		case when hcmref.corr_gid is not null then 1 else 0 end as is_system
	from dmrefs__correspondents cor 
	join (select mnames.link_id, cor_names.*, row_number() over (partition by link_id order by link_id asc) as rn
		from  dmrefs__correspondents_names mnames
		join dmrefs__corr_name cor_names on cor_names.id = mnames.names
	) names on names.link_id = cor.id and names.rn = 1
	left join (
		select t.*, row_number() over (partition by link_id order by link_id, codetype asc, is_main desc) as rn
		from (
	 	select cn.link_id, addr.idx, addr.country, names.city, names.street, names.building, names.room, adt.codetype, addr.is_main
	 	from dmrefs__correspondents_addresses cn 
	 	join dmrefs__corr_address addr on addr.id = cn.addresses 
	 	join (select id, case when code = '0001' then 0 else 1 end as codetype from dmrefs__corr_addr_types) adt on adt.id = addr.type_id
	 	left join dmrefs__corr_address_versions av on av.link_id = addr.id
	 	left join dmrefs__corr_addr_version names on names.id = av.versions
		) t
	) addresses on addresses.link_id = cor.id and addresses.rn = 1 
	left join dmhcm_corr_xrefs hcmref on hcmref.corr_gid = cor.gid
	where cor.is_current = 1
);
```

## groovy

### Выборка документов для группы доступа
**Задача:** вывести все документы, доступных пользователям с группой доступа X.

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

### Отчет по нормативным МПА на регистрации
**Задача:** вывести все нормативные МПА в статусе "Регистрация" с ссылками на связанные документы.

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

### Отчет по рассылке документа
**Задача:** для конкретного документа вывести историю рассылки (отправитель, адресат, ознакомлен/не знакомлен, дата ознакомления).

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