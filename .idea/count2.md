Для расширения API разработчика я создам механизм подсчета элементов на стороне базы данных по группам любого уровня вложенности, используя расширенный `FieldMappingRegistry` и метод репозитория. Итоговая структура данных будет иметь тип `Map<RegistryCountEnum, List<Pair<E, Long>>>`. 

Полный код решения, включая реализацию подсчета, будет соответствовать лучшим практикам и принципам SOLID.

---

### Полный Код

#### 1. `RegistryCountEnum`
```java
public enum RegistryCountEnum {
    STATUS_COUNT, // Пример группировки по статусу
    RECIPIENT_COUNT, // Пример группировки по получателю
    PAYMENT_TYPE_COUNT; // Пример группировки по типу платежа

    // Можно добавлять другие типы подсчетов.
}
```

---

#### 2. Расширение `FieldMappingRegistry`
Мы добавляем в `FieldMappingRegistry` поддержку группировок. Каждому `RegistryCountEnum` сопоставляются пути (поля) для группировки. 

```java
public class FieldMappingRegistry {

    private static final Map<RegistryCountEnum, BiFunction<Root<PaymentAnalyticEntity>, CriteriaBuilder, List<Expression<?>>>> GROUP_BY_MAPPING = new EnumMap<>(RegistryCountEnum.class);

    static {
        // Группировка по статусу
        GROUP_BY_MAPPING.put(RegistryCountEnum.STATUS_COUNT, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.status)));

        // Группировка по получателю
        GROUP_BY_MAPPING.put(RegistryCountEnum.RECIPIENT_COUNT, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.recipientName)));

        // Группировка по типу платежа
        GROUP_BY_MAPPING.put(RegistryCountEnum.PAYMENT_TYPE_COUNT, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.paymentType)));

        // Дополнительные группировки можно добавлять здесь.
    }

    public static List<Expression<?>> getGroupByPaths(@NonNull RegistryCountEnum countEnum,
                                                      @NonNull Root<PaymentAnalyticEntity> root,
                                                      @NonNull CriteriaBuilder cb) {
        BiFunction<Root<PaymentAnalyticEntity>, CriteriaBuilder, List<Expression<?>>> mappingFunction = GROUP_BY_MAPPING.get(countEnum);
        if (mappingFunction == null) {
            throw new IllegalArgumentException("No mapping found for RegistryCountEnum: " + countEnum);
        }
        return mappingFunction.apply(root, cb);
    }
}
```

---

#### 3. `GroupCountService`
Этот сервис использует расширенный `FieldMappingRegistry` для выполнения группировки и подсчета на стороне базы данных. Он возвращает `Map<RegistryCountEnum, List<Pair<E, Long>>>`.

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import lombok.NonNull;
import jakarta.persistence.EntityManager;
import jakarta.persistence.criteria.*;

import java.util.*;
import java.util.stream.Collectors;

@Service
public class GroupCountService {

    private final EntityManager entityManager;

    public GroupCountService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional(readOnly = true)
    public <E> Map<RegistryCountEnum, List<Pair<E, Long>>> countByGroup(
            @NonNull RegistryCountEnum countEnum,
            @NonNull Specification<PaymentAnalyticEntity> specification) {

        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Tuple> query = cb.createTupleQuery();
        Root<PaymentAnalyticEntity> root = query.from(PaymentAnalyticEntity.class);

        // Получаем поля для группировки
        List<Expression<?>> groupByPaths = FieldMappingRegistry.getGroupByPaths(countEnum, root, cb);

        // Применяем спецификацию фильтрации
        Predicate predicate = specification.toPredicate(root, query, cb);
        if (predicate != null) {
            query.where(predicate);
        }

        // Добавляем поля группировки и подсчет
        List<Selection<?>> selections = new ArrayList<>(groupByPaths);
        selections.add(cb.count(root).alias("count"));

        query.multiselect(selections);
        query.groupBy(groupByPaths);

        // Выполняем запрос
        List<Tuple> results = entityManager.createQuery(query).getResultList();

        // Обрабатываем результаты
        return results.stream()
                .collect(Collectors.groupingBy(
                        tuple -> countEnum,
                        Collectors.mapping(
                                tuple -> {
                                    @SuppressWarnings("unchecked")
                                    E groupValue = (E) tuple.get(0); // Первая колонка - значение группы
                                    Long count = ((Number) tuple.get("count")).longValue(); // Колонка count
                                    return Pair.of(groupValue, count);
                                },
                                Collectors.toUnmodifiableList()
                        )
                ));
    }
}
```

---

#### 4. Пример Использования в Сервисе

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import lombok.NonNull;

import java.util.List;
import java.util.Map;

@Service
public class PaymentAnalyticService {

    private final PaymentAnalyticRepository repository;
    private final GroupCountService groupCountService;

    public PaymentAnalyticService(PaymentAnalyticRepository repository, GroupCountService groupCountService) {
        this.repository = repository;
        this.groupCountService = groupCountService;
    }

    @Transactional(readOnly = true)
    public Map<RegistryCountEnum, List<Pair<String, Long>>> getCountByStatusAndRecipient(
            @NonNull PFLoanFundsRqRegistryFilterDto filterDto) {

        Specification<PaymentAnalyticEntity> specification = new PFLoanFundsRqRegistryPaymentAnalyticFilterSpecification(filterDto);

        // Подсчет по статусу
        Map<RegistryCountEnum, List<Pair<String, Long>>> statusCounts = groupCountService.countByGroup(
                RegistryCountEnum.STATUS_COUNT, specification);

        // Подсчет по получателю
        Map<RegistryCountEnum, List<Pair<String, Long>>> recipientCounts = groupCountService.countByGroup(
                RegistryCountEnum.RECIPIENT_COUNT, specification);

        // Объединяем результаты
        statusCounts.putAll(recipientCounts);
        return statusCounts;
    }
}
```

---

#### 5. Расширение Репозитория

Если нужно использовать функционал на базе QueryDSL, добавим метод в кастомный репозиторий, например, для подсчета через `groupBy`.

```java
import com.querydsl.core.types.dsl.Expressions;
import com.querydsl.jpa.impl.JPAQueryFactory;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public class PaymentAnalyticCustomRepository {

    private final JPAQueryFactory queryFactory;

    public PaymentAnalyticCustomRepository(JPAQueryFactory queryFactory) {
        this.queryFactory = queryFactory;
    }

    public List<Tuple> countByGroup(QueryDslPath path, Predicate predicate, Path<?> groupBy) {
        return queryFactory
                .select(path, Expressions.numberPath(Long.class, "count"))
                .from(path)
                .where(predicate)
                .groupBy(groupBy)
                .fetch();
    }
}
```

---

### Преимущества Решения

1. **Централизация:** Вся логика сопоставления уровней вложенности для группировки и подсчета сосредоточена в `FieldMappingRegistry`.
2. **Переиспользование:** Механизм подсчета подходит для любых вложенных уровней данных.
3. **Удобство:** Простое и гибкое API для подсчета элементов, используя спецификации.
4. **Расширяемость:** Легко добавить новые уровни группировки, расширив `RegistryCountEnum` и `FieldMappingRegistry`.

Данное решение позволяет эффективно работать с группировками и подсчетами, поддерживает чистую архитектуру и соблюдает принципы SOLID.




Понимаю ваши требования. Ниже представлен полностью переработанный класс `FieldMappingRegistry`, который централизует логику сопоставления `RegistryCodes` с соответствующими путями (полями) сущности `PaymentAnalyticEntity`. Этот класс предназначен для использования как в спецификациях фильтрации, так и в спецификациях сортировки, обеспечивая переиспользование и поддержание порядка путей при наличии нескольких полей для одного кода.

### Полный Код `FieldMappingRegistry`

```java
// --------------------------------------
// FieldMappingRegistry.java
// --------------------------------------

import jakarta.persistence.criteria.*;
import org.springframework.lang.NonNull;

import java.util.*;
import java.util.function.BiFunction;

/**
 * FieldMappingRegistry централизует сопоставление между RegistryCodes и списками путей (Expressions)
 * сущности PaymentAnalyticEntity. Он используется как для фильтрации, так и для сортировки, обеспечивая
 * согласованность и переиспользование логики.
 */
public class FieldMappingRegistry {

    /**
     * Map, содержащая соответствия между RegistryCodes и функциями, генерирующими списки Expressions.
     * Для каждого RegistryCode определяется BiFunction, принимающая Root и CriteriaBuilder,
     * возвращающая список путей (Expressions) в заданном порядке.
     */
    private static final Map<RegistryCodes, BiFunction<Root<PaymentAnalyticEntity>, CriteriaBuilder, List<Expression<?>>>> MAPPING = new EnumMap<>(RegistryCodes.class);

    static {
        // Инициализация соответствий

        // Простые поля
        MAPPING.put(RegistryCodes.NUMBER, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.number)));

        MAPPING.put(RegistryCodes.LOAN_FUNDS_REQUEST_STATUS, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.status)));

        MAPPING.put(RegistryCodes.DATE, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.date)));

        MAPPING.put(RegistryCodes.FUNDS_TYPE, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.fundsType)));

        MAPPING.put(RegistryCodes.TRANCHE_ISSUE_DATE, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.trancheIssueDate)));

        MAPPING.put(RegistryCodes.AMOUNT, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.amount)));

        MAPPING.put(RegistryCodes.PAYMENT_PURPOSE, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.paymentPurpose)));

        MAPPING.put(RegistryCodes.RECIPIENT_NAME, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.recipientName)));

        MAPPING.put(RegistryCodes.RECIPIENT_ACCOUNT, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.recipientAccount)));

        MAPPING.put(RegistryCodes.PAYMENT_TYPE, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.paymentType)));

        // Поля с несколькими путями
        MAPPING.put(RegistryCodes.CREDIT_AGREEMENT, (root, cb) -> List.of(
                root.get(PaymentAnalyticEntity_.creditAgreementNumber),
                root.get(PaymentAnalyticEntity_.creditAgreementDate)
        ));

        MAPPING.put(RegistryCodes.PAYER_ACCOUNT, (root, cb) -> List.of(
                root.get(PaymentAnalyticEntity_.payerAccount),
                root.get(PaymentAnalyticEntity_.obcAccountFlag)
        ));

        MAPPING.put(RegistryCodes.DVRU, (root, cb) -> List.of(
                root.get(PaymentAnalyticEntity_.dvruNumber),
                root.get(PaymentAnalyticEntity_.dvruDate)
        ));

        // Комплексные пути с Join
        MAPPING.put(RegistryCodes.SSR_ARTICLE, (root, cb) -> {
            // Join: paymentObjects -> paymentObjectSsr -> paymentSsrArticles -> code
            Join<PaymentAnalyticEntity, PaymentObjectEntity> paymentObjectJoin = root.join(PaymentAnalyticEntity_.paymentObjects, JoinType.LEFT);
            Join<PaymentObjectEntity, PaymentObjectSsrEntity> paymentObjectSsrJoin = paymentObjectJoin.join(PaymentObjectEntity_.paymentObjectSsr, JoinType.LEFT);
            Join<PaymentObjectSsrEntity, PaymentSsrArticleEntity> paymentSsrArticleJoin = paymentObjectSsrJoin.join(PaymentObjectSsrEntity_.paymentSsrArticles, JoinType.LEFT);
            return List.of(paymentSsrArticleJoin.get(PaymentSsrArticleEntity_.code));
        });

        MAPPING.put(RegistryCodes.PAYMENT_OBJECT, (root, cb) -> {
            // Join: paymentObjects -> name, projectName
            Join<PaymentAnalyticEntity, PaymentObjectEntity> paymentObjectJoin = root.join(PaymentAnalyticEntity_.paymentObjects, JoinType.LEFT);
            return List.of(
                    paymentObjectJoin.get(PaymentObjectEntity_.name),
                    paymentObjectJoin.get(PaymentObjectEntity_.projectName)
            );
        });

        // PAYMENT_ORDER_STATUS - предположим, что это поле status
        MAPPING.put(RegistryCodes.PAYMENT_ORDER_STATUS, (root, cb) -> List.of(root.get(PaymentAnalyticEntity_.status)));
    }

    /**
     * Получает список Expressions (путей) для заданного RegistryCode.
     *
     * @param code  Код поля из RegistryCodes.
     * @param root  Root сущности PaymentAnalyticEntity.
     * @param cb    CriteriaBuilder для построения выражений.
     * @return Список Expressions, соответствующих коду, в порядке объявления.
     * @throws IllegalArgumentException Если для кода не найдено сопоставление.
     */
    @NonNull
    public static List<Expression<?>> getPaths(@NonNull RegistryCodes code,
                                              @NonNull Root<PaymentAnalyticEntity> root,
                                              @NonNull CriteriaBuilder cb) {
        BiFunction<Root<PaymentAnalyticEntity>, CriteriaBuilder, List<Expression<?>>> mappingFunction = MAPPING.get(code);
        if (mappingFunction == null) {
            throw new IllegalArgumentException("No mapping found for RegistryCode: " + code);
        }
        return mappingFunction.apply(root, cb);
    }
}
```

### Объяснение Реализации

1. **Централизация Логики Сопоставления:**
   - Класс `FieldMappingRegistry` содержит статическую карту `MAPPING`, которая сопоставляет каждый `RegistryCodes` с соответствующей функцией.
   - Каждая функция принимает `Root<PaymentAnalyticEntity>` и `CriteriaBuilder` и возвращает список `Expression<?>`, представляющих пути к полям, соответствующим данному коду.

2. **Обработка Множественных Полей:**
   - Для кодов, соответствующих нескольким полям (например, `CREDIT_AGREEMENT`, `PAYER_ACCOUNT`, `DVRU`), список `Expression<?>` содержит все соответствующие поля в порядке их объявления.
   - Это обеспечивает правильный порядок сортировки и фильтрации при наличии нескольких полей.

3. **Комплексные Пути с Join:**
   - Для кодов, требующих доступа к вложенным сущностям (например, `SSR_ARTICLE`, `PAYMENT_OBJECT`), функции выполняют необходимые `join` и возвращают пути к вложенным полям.
   - Это позволяет использовать сложные структуры данных без дублирования логики в спецификациях фильтрации и сортировки.

4. **Метод `getPaths`:**
   - Метод `getPaths` предоставляет список `Expression<?>` для заданного `RegistryCodes`.
   - Если код не имеет сопоставления, выбрасывается `IllegalArgumentException`, что помогает в рантайм обнаруживать некорректные или неподдерживаемые коды.

### Использование `FieldMappingRegistry` в Спецификациях

Теперь, когда `FieldMappingRegistry` полностью реализован, его можно использовать как в спецификациях фильтрации, так и в спецификациях сортировки. Ниже приведены обновленные классы `PFLoanFundsRqRegistryPaymentAnalyticFilterSpecification` и `PFLoanFundsRqRegistryPaymentAnalyticSortingSpecification`, демонстрирующие, как использовать `FieldMappingRegistry` для получения путей.

#### Обновленный `PFLoanFundsRqRegistryPaymentAnalyticFilterSpecification`

```java
import jakarta.persistence.criteria.*;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.lang.Nullable;
import lombok.NonNull;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

public class PFLoanFundsRqRegistryPaymentAnalyticFilterSpecification implements Specification<PaymentAnalyticEntity> {

    private static final String INN_MAYBE_REGEX = "\\d{1,12}";
    
    @NonNull
    private final PFLoanFundsRqRegistryFilterDto filter;

    public PFLoanFundsRqRegistryPaymentAnalyticFilterSpecification(@NonNull PFLoanFundsRqRegistryFilterDto filter) {
        this.filter = filter;
    }

    @Override
    public Predicate toPredicate(@NonNull Root<PaymentAnalyticEntity> root,
                                 @NonNull CriteriaQuery<?> query,
                                 @NonNull CriteriaBuilder cb) {
        List<Predicate> predicates = new ArrayList<>();

        // Пример фильтрации с использованием FieldMappingRegistry
        // Применяем фильтры на основе RegistryCodes и соответствующих путей

        // Фильтр по статусам запроса на получение средств
        if (filter.getLoanFundsRequestStatuses() != null && !filter.getLoanFundsRequestStatuses().isEmpty()) {
            Set<PaymentAnalyticStatus> statuses = filter.getLoanFundsRequestStatuses().stream()
                    .map(PaymentAnalyticStatus::of)
                    .collect(Collectors.toUnmodifiableSet());
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.LOAN_FUNDS_REQUEST_STATUS, root, cb);
            predicates.add(cb.in(paths.get(0)).value(statuses));
        } else {
            // Если статусы не указаны, берем все возможные статусы
            predicates.add(cb.in(root.get(PaymentAnalyticEntity_.status)).value(Set.of(PaymentAnalyticStatus.values())));
        }

        // Фильтр по номеру
        if (filter.getNumber() != null && !filter.getNumber().isEmpty()) {
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.NUMBER, root, cb);
            Expression<String> numberAsString = cb.function("CAST", String.class, paths.get(0));
            predicates.add(cb.like(numberAsString, filter.getNumber() + "%"));
        }

        // Фильтр по дате выдачи
        if (filter.getLoanFundsRequestDate() != null) {
            if (filter.getLoanFundsRequestDate().getFrom() != null) {
                List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.DATE, root, cb);
                predicates.add(cb.greaterThanOrEqualTo(paths.get(0), filter.getLoanFundsRequestDate().getFrom()));
            }
            if (filter.getLoanFundsRequestDate().getTo() != null) {
                List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.DATE, root, cb);
                predicates.add(cb.lessThanOrEqualTo(paths.get(0), filter.getLoanFundsRequestDate().getTo()));
            }
        }

        // Фильтр по payerAccounts
        if (filter.getPayerAccounts() != null && !filter.getPayerAccounts().isEmpty()) {
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.PAYER_ACCOUNT, root, cb);
            predicates.add(cb.or(
                    paths.stream()
                         .map(cb::in)
                         .toArray(Predicate[]::new)
            ));
        }

        // Фильтр по recipientAccounts
        if (filter.getRecipientAccounts() != null && !filter.getRecipientAccounts().isEmpty()) {
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.RECIPIENT_ACCOUNT, root, cb);
            predicates.add(cb.in(paths.get(0)).value(filter.getRecipientAccounts()));
        }

        // Фильтр по диапазону суммы
        if (filter.getAmountFrom() != null) {
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.AMOUNT, root, cb);
            predicates.add(cb.greaterThanOrEqualTo((Expression<BigDecimal>) paths.get(0), filter.getAmountFrom()));
        }
        if (filter.getAmountTo() != null) {
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.AMOUNT, root, cb);
            predicates.add(cb.lessThanOrEqualTo((Expression<BigDecimal>) paths.get(0), filter.getAmountTo()));
        }

        // Фильтр по цели платежа
        if (filter.getPaymentPurposeSearch() != null && !filter.getPaymentPurposeSearch().isEmpty()) {
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.PAYMENT_PURPOSE, root, cb);
            predicates.add(cb.like(cb.lower((Expression<String>) paths.get(0)),
                    filter.getPaymentPurposeSearch().toLowerCase() + "%"));
        }

        // Фильтр по идентификаторам кредитных соглашений
        if (filter.getCreditAgreementIds() != null && !filter.getCreditAgreementIds().isEmpty()) {
            Set<UUID> creditAgreementIds = filter.getCreditAgreementIds().stream()
                    .map(UUID::fromString)
                    .collect(Collectors.toUnmodifiableSet());
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.CREDIT_AGREEMENT, root, cb);
            predicates.add(cb.and(
                    cb.in(paths.get(0)).value(creditAgreementIds),
                    cb.in(paths.get(1)).value(creditAgreementIds) // Предположим, что оба поля связаны с creditAgreementId
            ));
        }

        // Фильтр по дате выпуска транша
        if (filter.getTrancheIssueDate() != null) {
            if (filter.getTrancheIssueDate().getFrom() != null) {
                List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.TRANCHE_ISSUE_DATE, root, cb);
                predicates.add(cb.greaterThanOrEqualTo(paths.get(0), filter.getTrancheIssueDate().getFrom()));
            }
            if (filter.getTrancheIssueDate().getTo() != null) {
                List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.TRANCHE_ISSUE_DATE, root, cb);
                predicates.add(cb.lessThanOrEqualTo(paths.get(0), filter.getTrancheIssueDate().getTo()));
            }
        }

        // Фильтр по имени или ИНН получателя
        if (filter.getRecipientNameOrInn() != null && !filter.getRecipientNameOrInn().isEmpty()) {
            String val = filter.getRecipientNameOrInn();
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.RECIPIENT_NAME, root, cb);
            Predicate namePredicate = cb.like(cb.lower((Expression<String>) paths.get(0)), val.toLowerCase() + "%");

            Predicate finalPredicate = namePredicate;

            if (val.matches(INN_MAYBE_REGEX)) {
                List<Expression<?>> innPaths = FieldMappingRegistry.getPaths(RegistryCodes.DVRU, root, cb);
                Predicate innPredicate = cb.like((Expression<String>) innPaths.get(0), val + "%");
                finalPredicate = cb.or(namePredicate, innPredicate);
            }

            predicates.add(finalPredicate);
        }

        // Фильтр по типу фондов
        if (filter.getFundsType() != null && !filter.getFundsType().isEmpty()) {
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.FUNDS_TYPE, root, cb);
            predicates.add(cb.equal((Expression<String>) paths.get(0), FundsType.of(filter.getFundsType())));
        }

        // Фильтр по идентификаторам DVRU RBS
        if (filter.getDvruRbsIds() != null && !filter.getDvruRbsIds().isEmpty()) {
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.DVRU, root, cb);
            predicates.add(cb.in(paths.get(1)).value(filter.getDvruRbsIds())); // dvruDate
        }

        // Фильтр по типу платежа
        if (filter.getPaymentType() != null && !filter.getPaymentType().isEmpty()) {
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.PAYMENT_TYPE, root, cb);
            predicates.add(cb.equal((Expression<String>) paths.get(0), PaymentType.of(filter.getPaymentType())));
        }

        // Фильтр по наличию платежного документа (paymentOrder)
        if (filter.getHasPaymentOrder() != null) {
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.PAYMENT_ORDER_STATUS, root, cb);
            if (filter.getHasPaymentOrder()) {
                predicates.add(cb.isNotNull(paths.get(0)));
            } else {
                predicates.add(cb.isNull(paths.get(0)));
            }
        }

        // Фильтр по идентификаторам LKS объектов платежа
        if (filter.getPaymentObjectLksIds() != null && !filter.getPaymentObjectLksIds().isEmpty()) {
            // Используем FieldMappingRegistry для PAYMENT_OBJECT
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.PAYMENT_OBJECT, root, cb);
            Join<PaymentAnalyticEntity, PaymentObjectEntity> paymentObjectJoin = root.join(PaymentAnalyticEntity_.paymentObjects, JoinType.LEFT);
            predicates.add(paymentObjectJoin.get(PaymentObjectEntity_.lksId).in(filter.getPaymentObjectLksIds()));
        }

        // Фильтр по коду статьи SSR
        if (filter.getSsrArticleCode() != null && !filter.getSsrArticleCode().isEmpty()) {
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.SSR_ARTICLE, root, cb);
            Join<PaymentAnalyticEntity, PaymentObjectEntity> paymentObjectJoin = root.join(PaymentAnalyticEntity_.paymentObjects, JoinType.LEFT);
            Join<PaymentObjectEntity, PaymentObjectSsrEntity> paymentObjectSsrJoin = paymentObjectJoin.join(PaymentObjectEntity_.paymentObjectSsr, JoinType.LEFT);
            Join<PaymentObjectSsrEntity, PaymentSsrArticleEntity> paymentSsrArticleJoin = paymentObjectSsrJoin.join(PaymentObjectSsrEntity_.paymentSsrArticles, JoinType.LEFT);
            predicates.add(cb.like(paymentSsrArticleJoin.get(PaymentSsrArticleEntity_.code), "%" + filter.getSsrArticleCode() + "%"));
        }

        return cb.and(predicates.toArray(new Predicate[0]));
    }
}
```

#### Обновленный `PFLoanFundsRqRegistryPaymentAnalyticSortingSpecification`

```java
import jakarta.persistence.criteria.*;
import org.springframework.data.jpa.domain.Specification;
import lombok.NonNull;
import java.util.ArrayList;
import java.util.List;

/**
 * Спецификация для сортировки PaymentAnalyticEntity на основе списка SortingRequestDto.
 * Использует FieldMappingRegistry для получения путей сортировки.
 */
public class PFLoanFundsRqRegistryPaymentAnalyticSortingSpecification implements Specification<PaymentAnalyticEntity> {

    @NonNull
    private final List<SortingRequestDto> sortingDtos;

    public PFLoanFundsRqRegistryPaymentAnalyticSortingSpecification(@NonNull List<SortingRequestDto> sortingDtos) {
        this.sortingDtos = sortingDtos;
    }

    @Override
    public Predicate toPredicate(@NonNull Root<PaymentAnalyticEntity> root,
                                 @NonNull CriteriaQuery<?> query,
                                 @NonNull CriteriaBuilder cb) {
        List<Order> orders = new ArrayList<>();

        if (!sortingDtos.isEmpty()) {
            for (SortingRequestDto dto : sortingDtos) {
                boolean descending = dto.getSortDescending();
                RegistryCodes code = RegistryCodes.of(dto.getSortField());

                // Получаем список путей для данного кода
                List<Expression<?>> paths = FieldMappingRegistry.getPaths(code, root, cb);

                // Применяем сортировку по каждому пути в порядке их объявления
                for (Expression<?> path : paths) {
                    orders.add(descending ? cb.desc(path) : cb.asc(path));
                }
            }
        } else {
            // Сортировка по умолчанию - по дате убыв.
            List<Expression<?>> paths = FieldMappingRegistry.getPaths(RegistryCodes.DATE, root, cb);
            if (!paths.isEmpty()) {
                orders.add(cb.desc(paths.get(0)));
            }
        }

        query.orderBy(orders);
        return null; // Сортировка не влияет на Predicate
    }
}
```

### Объяснение Использования `FieldMappingRegistry` в Спецификациях

1. **Фильтрация (`PFLoanFundsRqRegistryPaymentAnalyticFilterSpecification`):**
   - При необходимости фильтрации по определенному `RegistryCodes`, спецификация вызывает `FieldMappingRegistry.getPaths(code, root, cb)` для получения списка соответствующих путей.
   - Затем, в зависимости от логики фильтрации (например, `IN`, `LIKE`, `EQUAL`), спецификация применяет условия к этим путям.
   - Если `RegistryCodes` соответствует нескольким путям, фильтрация может комбинировать условия для всех этих путей (например, используя `AND` или `OR`), в зависимости от бизнес-логики.

2. **Сортировка (`PFLoanFundsRqRegistryPaymentAnalyticSortingSpecification`):**
   - При необходимости сортировки по определенному `RegistryCodes`, спецификация вызывает `FieldMappingRegistry.getPaths(code, root, cb)` для получения списка соответствующих путей.
   - Затем, в порядке объявления путей, добавляет `Order` в `CriteriaQuery` для каждого пути, сохраняя заданный порядок сортировки.

3. **Преимущества:**
   - **Централизация:** Вся логика сопоставления кодов с путями находится в одном месте, что облегчает поддержку и расширение.
   - **Переиспользование:** Одни и те же пути используются как для фильтрации, так и для сортировки, предотвращая дублирование кода.
   - **Согласованность:** Обеспечивается согласованность между фильтрацией и сортировкой, особенно при наличии нескольких путей для одного кода.
   - **Гибкость:** Легко добавлять новые `RegistryCodes` или изменять существующие сопоставления без изменения спецификаций.

### Пример Использования в Сервисном Слое

Для полноты картины приведем пример, как эти спецификации используются в сервисе для фильтрации и сортировки данных.

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import lombok.NonNull;
import org.springframework.lang.Nullable;
import java.util.List;

@Service
public class PaymentAnalyticService {

    private final PaymentAnalyticRepository repository;
    private final FilterService filterService;
    private final SortingService sortingService;

    public PaymentAnalyticService(PaymentAnalyticRepository repository, FilterService filterService, SortingService sortingService) {
        this.repository = repository;
        this.filterService = filterService;
        this.sortingService = sortingService;
    }

    /**
     * Получает отфильтрованные и отсортированные данные.
     *
     * @param pageable      Объект Pageable, содержащий информацию о пагинации и сортировке.
     * @param filterDto     DTO, содержащий параметры фильтрации.
     * @param sortingFields Список SortingRequestDto, содержащий параметры сортировки.
     * @return Страница PaymentAnalyticEntity, соответствующая фильтрам и сортировкам.
     */
    public Page<PaymentAnalyticEntity> getFilteredAndSortedData(
            @NonNull Pageable pageable,
            @Nullable PFLoanFundsRqRegistryFilterDto filterDto,
            @Nullable List<SortingRequestDto> sortingFields) {

        Specification<PaymentAnalyticEntity> filterSpec = null;
        if (filterDto != null) {
            filterSpec = filterService.applyFilter(filterDto);
        }

        Specification<PaymentAnalyticEntity> sortingSpec = null;
        if (sortingFields != null && !sortingFields.isEmpty()) {
            sortingSpec = sortingService.applySorting(sortingFields);
        }

        SpecificationBuilder<PaymentAnalyticEntity> builder = new SpecificationBuilder<>();
        builder.add(filterSpec);

        // Если в pageable отсутствует сортировка, применяем нашу спецификацию сортировки
        if (pageable.getSort().isUnsorted() && sortingSpec != null) {
            builder.add(sortingSpec);
        }

        Specification<PaymentAnalyticEntity> finalSpec = builder.build();
        return repository.findAll(finalSpec, pageable);
    }
}
```

### Обновленный `FilterService` и `SortingService`

#### `FilterService.java`

```java
import org.springframework.stereotype.Service;
import lombok.NonNull;
import org.springframework.lang.Nullable;

@Service
public class FilterService {
    /**
     * Применяет фильтр на основе PFLoanFundsRqRegistryFilterDto.
     *
     * @param filterDto DTO с параметрами фильтрации.
     * @return Спецификация для фильтрации.
     */
    public Specification<PaymentAnalyticEntity> applyFilter(@NonNull PFLoanFundsRqRegistryFilterDto filterDto) {
        return new PFLoanFundsRqRegistryPaymentAnalyticFilterSpecification(filterDto);
    }
}
```

#### `SortingService.java`

```java
import org.springframework.stereotype.Service;
import lombok.NonNull;
import org.springframework.lang.Nullable;
import java.util.List;

@Service
public class SortingService {
    /**
     * Применяет сортировку на основе списка SortingRequestDto.
     *
     * @param sortingDtos Список DTO с параметрами сортировки.
     * @return Спецификация для сортировки.
     */
    public Specification<PaymentAnalyticEntity> applySorting(@NonNull List<SortingRequestDto> sortingDtos) {
        return new PFLoanFundsRqRegistryPaymentAnalyticSortingSpecification(sortingDtos);
    }
}
```

### Обновленный Репозиторий

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.repository.CrudRepository;
import org.springframework.lang.Nullable;
import java.util.UUID;

public interface PaymentAnalyticRepository extends CrudRepository<PaymentAnalyticEntity, UUID>, JpaSpecificationExecutor<PaymentAnalyticEntity> {

    @EntityGraph(attributePaths = {
            "paymentObjects",
            "paymentObjects.paymentObjectSsr",
            "paymentObjects.paymentObjectSsr.paymentSsrArticles"
    })
    Page<PaymentAnalyticEntity> findAll(@Nullable Specification<PaymentAnalyticEntity> spec, @NonNull Pageable pageable);
}
```

### Заключение

С помощью полностью переработанного класса `FieldMappingRegistry` мы централизовали логику сопоставления `RegistryCodes` с путями сущности `PaymentAnalyticEntity`. Это обеспечивает:

- **Переиспользование** логики в фильтрации и сортировке.
- **Поддержание порядка** путей для кодов, соответствующих нескольким полям.
- **Удобство расширения**: добавление новых кодов или изменение существующих сводится к модификации только `FieldMappingRegistry`.

Теперь ваши спецификации фильтрации и сортировки используют одно и то же централизованное сопоставление путей, что соответствует принципам DRY и SOLID, обеспечивая чистую и поддерживаемую архитектуру.
