```java

import jakarta.persistence.criteria.*;
import org.springframework.lang.NonNull;

import java.util.*;
import java.util.function.BiFunction;

/**
 * FieldMappingRegistry централизует сопоставление между RegistryCodes и списками путей (Expressions)
 * сущности PaymentAnalyticEntity. Он используется как для фильтрации, так и для сортировки, обеспечивая
 * согласованность и переиспользование логики.
 *
 * @author Владислав Турукин
 */
public class FieldMappingRegistry {

  /**
   * Константы для управления порядком сортировки и фильтрации.
   */
  public static final String ORDER_FIRST = "ORDER_FIRST";
  public static final String ORDER_SECOND = "ORDER_SECOND";
  public static final String ORDER_THIRD = "ORDER_THIRD";

  /**
   * Map, содержащая соответствия между RegistryCodes и функциями, генерирующими списки Expressions.
   * Для каждого RegistryCode определяется BiFunction, принимающая Root и CriteriaBuilder,
   * возвращающая список путей (Expressions) в заданном порядке.
   */
  private static final Map<RegistryCodes, BiFunction<Root<PaymentAnalyticEntity>, CriteriaBuilder, List<Expression<?>>>> MAPPING = new LinkedHashMap<>();

  static {
    // Простые поля
    MAPPING.put(RegistryCodes.NUMBER, (root, cb) ->
            MetamodelUtility.getExpressionsForAttribute(PaymentAnalyticEntity_.number, root));
    MAPPING.put(RegistryCodes.LOAN_FUNDS_REQUEST_STATUS, (root, cb) ->
            MetamodelUtility.getExpressionsForAttribute(PaymentAnalyticEntity_.status, root));
    MAPPING.put(RegistryCodes.DATE, (root, cb) ->
            MetamodelUtility.getExpressionsForAttribute(PaymentAnalyticEntity_.date, root));
    MAPPING.put(RegistryCodes.FUNDS_TYPE, (root, cb) ->
            MetamodelUtility.getExpressionsForAttribute(PaymentAnalyticEntity_.fundsType, root));
    MAPPING.put(RegistryCodes.TRANCHE_ISSUE_DATE, (root, cb) ->
            MetamodelUtility.getExpressionsForAttribute(PaymentAnalyticEntity_.trancheIssueDate, root));
    MAPPING.put(RegistryCodes.AMOUNT, (root, cb) ->
            MetamodelUtility.getExpressionsForAttribute(PaymentAnalyticEntity_.amount, root));
    MAPPING.put(RegistryCodes.PAYMENT_PURPOSE, (root, cb) ->
            MetamodelUtility.getExpressionsForAttribute(PaymentAnalyticEntity_.paymentPurpose, root));
    MAPPING.put(RegistryCodes.RECIPIENT_NAME, (root, cb) ->
            MetamodelUtility.getExpressionsForAttribute(PaymentAnalyticEntity_.recipientName, root));
    MAPPING.put(RegistryCodes.RECIPIENT_ACCOUNT, (root, cb) ->
            MetamodelUtility.getExpressionsForAttribute(PaymentAnalyticEntity_.recipientAccount, root));
    MAPPING.put(RegistryCodes.PAYMENT_TYPE, (root, cb) ->
            MetamodelUtility.getExpressionsForAttribute(PaymentAnalyticEntity_.paymentType, root));

    // Поля с несколькими путями
    MAPPING.put(RegistryCodes.CREDIT_AGREEMENT, (root, cb) ->
            MetamodelUtility.createCompositePaths(root,
                    PaymentAnalyticEntity_.creditAgreementNumber,
                    PaymentAnalyticEntity_.creditAgreementDate));
    MAPPING.put(RegistryCodes.PAYER_ACCOUNT, (root, cb) ->
            MetamodelUtility.createCompositePaths(root,
                    PaymentAnalyticEntity_.payerAccount,
                    PaymentAnalyticEntity_.obcAccountFlag));
    MAPPING.put(RegistryCodes.DVRU, (root, cb) ->
            MetamodelUtility.createCompositePaths(root,
                    PaymentAnalyticEntity_.dvruNumber,
                    PaymentAnalyticEntity_.dvruDate));

    // Комплексные пути с Join
    MAPPING.put(RegistryCodes.PAYMENT_OBJECT, (root, cb) -> {
      Join<PaymentAnalyticEntity, PaymentObjectEntity> paymentObjectJoin =
              MetamodelUtility.createJoin(root, PaymentAnalyticEntity_.paymentObjects);
      return MetamodelUtility.createCompositePaths(paymentObjectJoin,
              PaymentObjectEntity_.name,
              PaymentObjectEntity_.projectName);
    });
    MAPPING.put(RegistryCodes.SSR_ARTICLE, (root, cb) -> {
      Join<PaymentAnalyticEntity, PaymentObjectEntity> paymentObjectJoin =
              MetamodelUtility.createJoin(root, PaymentAnalyticEntity_.paymentObjects);
      Join<PaymentObjectEntity, PaymentObjectSsrEntity> paymentObjectSsrJoin =
              MetamodelUtility.createJoin(paymentObjectJoin, PaymentObjectEntity_.paymentObjectSsr);
      Join<PaymentObjectSsrEntity, PaymentSsrArticleEntity> paymentSsrArticleJoin =
              MetamodelUtility.createJoin(paymentObjectSsrJoin, PaymentObjectSsrEntity_.paymentSsrArticles);
      return MetamodelUtility.getExpressionsForAttribute(PaymentSsrArticleEntity_.code, paymentSsrArticleJoin);
    });

    // PAYMENT_ORDER_STATUS
    MAPPING.put(RegistryCodes.PAYMENT_ORDER_STATUS, (root, cb) ->
            MetamodelUtility.getExpressionsForAttribute(PaymentAnalyticEntity_.status, root));
  }

  /**
   * Получает список Expressions (путей) для заданного RegistryCode.
   *
   * @param code  Код поля из {@link RegistryCodes}.
   * @param root  {@link Root} сущности PaymentAnalyticEntity.
   * @param cb    {@link CriteriaBuilder} для построения выражений.
   * @return Список {@link Expression}, соответствующих коду, в порядке объявления.
   * @throws IllegalArgumentException Если для кода не найдено сопоставление.
   */
  @NonNull
  public static List<Expression<?>> getPaths(@NonNull RegistryCodes code,
                                             @NonNull Root<PaymentAnalyticEntity> root,
                                             @NonNull CriteriaBuilder cb) {
    BiFunction<Root<PaymentAnalyticEntity>, CriteriaBuilder, List<Expression<?>>> mappingFunction = MAPPING.get(code);
    if (mappingFunction == null) {
      throw new IllegalArgumentException("Не найдено сопоставление для кода: " + code);
    }
    return mappingFunction.apply(root, cb);
  }
}


```
```java
import jakarta.persistence.criteria.*;
import org.springframework.lang.NonNull;

import java.util.EnumMap;
import java.util.List;
import java.util.Map;
import java.util.function.BiFunction;

/**
 * CountGroupMappingRegistry предоставляет группировки для операций подсчета
 * на основе различных полей сущности PaymentAnalyticEntity.
 *
 * @author Владислав Турукин
 */
public class CountGroupMappingRegistry {

    private static final Map<RegistryCountEnum, BiFunction<Root<PaymentAnalyticEntity>, CriteriaBuilder, List<Expression<?>>>> GROUP_BY_MAPPING = new EnumMap<>(RegistryCountEnum.class);

    static {
        // Группировка по статусу
        GROUP_BY_MAPPING.put(RegistryCountEnum.STATUS_COUNT, (root, cb) ->
                List.of(root.get(PaymentAnalyticEntity_.status)));

        // Группировка по получателю
        GROUP_BY_MAPPING.put(RegistryCountEnum.RECIPIENT_COUNT, (root, cb) ->
                List.of(root.get(PaymentAnalyticEntity_.recipientName)));

        // Группировка по типу платежа
        GROUP_BY_MAPPING.put(RegistryCountEnum.PAYMENT_TYPE_COUNT, (root, cb) ->
                List.of(root.get(PaymentAnalyticEntity_.paymentType)));

        // Дополнительные группировки можно добавлять здесь.
    }

    /**
     * Возвращает пути для группировки на основе заданного типа подсчета.
     *
     * @param countEnum Тип подсчета из {@link RegistryCountEnum}.
     * @param root      Корневой объект сущности PaymentAnalyticEntity.
     * @param cb        Объект CriteriaBuilder для построения выражений.
     * @return Список путей ({@link Expression}) для группировки.
     * @throws IllegalArgumentException Если не найдено сопоставление для заданного типа подсчета.
     */
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
```java
import jakarta.persistence.criteria.*;
import org.springframework.lang.NonNull;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;

/**
 * MetamodelUtility предоставляет вспомогательные методы для работы с JPA Criteria API, включая создание цепочек JOIN,
 * получение выражений из метамодели и построение сложных путей для фильтрации и сортировки.
 *
 * Этот класс разработан для упрощения работы с метамоделями Hibernate и обеспечения строгой типизации.
 *
 * @author Владислав Турукин
 */
public class MetamodelUtility {

  /**
   * Создает цепочку JOIN на основе заданного списка метаполей.
   *
   * @param root       {@link Root} сущности.
   * @param attributes Список метаполей, описывающих цепочку JOIN.
   * @return Последний {@link Join} в цепочке.
   */
  public static Join<?, ?> createJoinChain(@NonNull Root<?> root, @NonNull List<SingularAttribute<?, ?>> attributes) {
    Join<?, ?> join = null;
    for (SingularAttribute<?, ?> attribute : attributes) {
      join = (join == null) ? root.join(attribute, JoinType.LEFT) : join.join(attribute, JoinType.LEFT);
    }
    return join;
  }

  /**
   * Возвращает неизменяемый список выражений для одиночного поля метамодели.
   *
   * @param attribute Поле метамодели.
   * @param root      {@link Root} сущности.
   * @param <T>       Тип данных поля.
   * @return Типизированный неизменяемый список {@link Expression} для указанного поля.
   */
  public static <T> List<Expression<T>> getExpressionsForAttribute(@NonNull SingularAttribute<?, T> attribute,
                                                                   @NonNull Root<?> root) {
    return Collections.unmodifiableList(List.of(root.get(attribute)));
  }

  /**
   * Создает неизменяемый список выражений для вложенных полей, используя цепочку JOIN.
   *
   * @param joinAttributes Список метаполей для построения цепочки JOIN.
   * @param finalAttribute Конечное поле, для которого возвращается выражение.
   * @param root           {@link Root} сущности.
   * @param <T>            Тип данных конечного поля.
   * @return Неизменяемый список {@link Expression} для вложенного поля.
   */
  public static <T> List<Expression<T>> getNestedExpressions(@NonNull List<SingularAttribute<?, ?>> joinAttributes,
                                                             @NonNull SingularAttribute<?, T> finalAttribute,
                                                             @NonNull Root<?> root) {
    Join<?, ?> join = createJoinChain(root, joinAttributes);
    return Collections.unmodifiableList(List.of(join.get(finalAttribute)));
  }

  /**
   * Создает неизменяемый список выражений для нескольких атрибутов.
   *
   * @param root       {@link Root} сущности.
   * @param attributes Атрибуты метамодели.
   * @param <T>        Тип данных атрибутов.
   * @return Неизменяемый список {@link Expression}.
   */
  @SafeVarargs
  public static <T> List<Expression<T>> createCompositePaths(@NonNull Root<?> root,
                                                             @NonNull SingularAttribute<?, T>... attributes) {
    return Collections.unmodifiableList(
            List.of(attributes).stream()
                    .map(root::get)
                    .toList()
    );
  }
}

```
```java
import org.springframework.data.jpa.domain.Specification;
import jakarta.persistence.criteria.*;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Абстрактный класс SortingService предоставляет общую функциональность для реализации сортировки.
 * Он может быть расширен для предоставления специализированных реализаций сортировки.
 *
 * @author Владислав Турукин
 * @param <T> Тип сущности, для которой применяется сортировка.
 */
public abstract class SortingService<T> {

  /**
   * Применяет сортировку к сущности на основе списка параметров сортировки.
   *
   * @param entityClass  Класс сущности.
   * @param sortingDtos  Список параметров сортировки.
   * @return Спецификация для применения сортировки.
   */
  public Specification<T> applySorting(Class<T> entityClass, List<SortingRequestDto> sortingDtos) {
    return (root, query, cb) -> {
      List<Order> orders = new ArrayList<>();

      if (sortingDtos != null && !sortingDtos.isEmpty()) {
        for (SortingRequestDto dto : sortingDtos) {
          RegistryCodes code = RegistryCodes.of(dto.getSortField());

          List<Expression<?>> expressions = FieldMappingRegistry.getPaths(code, root, cb);
          if (expressions.isEmpty()) {
            throw new IllegalArgumentException("Не найдено сопоставление для кода: " + code);
          }

          for (Expression<?> expression : expressions) {
            orders.add(dto.getSortDescending() ? cb.desc(expression) : cb.asc(expression));
          }
        }
      } else {
        List<Expression<?>> defaultExpression = FieldMappingRegistry.getPaths(RegistryCodes.DATE, root, cb);
        if (!defaultExpression.isEmpty()) {
          orders.add(cb.desc(defaultExpression.get(0)));
        }
      }

      query.orderBy(Collections.unmodifiableList(orders));
      return null;
    };
  }

  /**
   * Метод для предоставления дополнительной логики сортировки.
   * Может быть переопределен в подклассах.
   *
   * @param root Корневой объект сущности.
   * @param query Объект запроса.
   * @param cb Критерий построения.
   */
  protected abstract void customizeSorting(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
}
```
```java
import org.springframework.data.jpa.domain.Specification;
import jakarta.persistence.criteria.*;

import java.util.List;

/**
 * FilterService предоставляет функциональность для применения фильтров к сущностям
 * на основе списков RegistryCodes.
 *
 * @author Владислав Турукин
 */
public class FilterService {

    /**
     * Применяет фильтры к сущности на основе списка RegistryCodes.
     *
     * @param entityClass Класс сущности.
     * @param fields      Список кодов полей для фильтрации.
     * @param <T>         Тип сущности.
     * @return Спецификация для применения фильтров.
     */
    public <T> Specification<T> applyFilter(Class<T> entityClass,
                                            List<RegistryCodes> fields) {
        return (root, query, cb) -> {
            Predicate predicate = cb.conjunction();

            for (RegistryCodes code : fields) {
                // Получаем пути через FieldMappingRegistry
                List<Expression<?>> expressions = FieldMappingRegistry.getPaths(code, root, cb);
                if (expressions.isEmpty()) {
                    throw new IllegalArgumentException("Не найдено сопоставление для кода: " + code);
                }

                for (Expression<?> expression : expressions) {
                    // Добавляем условие isNotNull для каждого выражения
                    predicate = cb.and(predicate, cb.isNotNull(expression));
                }
            }
            return predicate;
        };
    }
}
```
```java
import org.springframework.data.jpa.domain.Specification;
import org.springframework.lang.Nullable;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * SpecificationBuilder предоставляет удобный способ создания цепочек спецификаций для фильтрации и сортировки сущностей.
 *
 * @param <T> Тип сущности.
 *
 */
public class SpecificationBuilder<T> {

    private final List<Specification<T>> specifications = new ArrayList<>();

    /**
     * Добавляет спецификацию в цепочку.
     *
     * @param spec Спецификация для добавления. Если null, спецификация игнорируется.
     * @return Текущий экземпляр SpecificationBuilder для цепочки вызовов.
     */
    public SpecificationBuilder<T> add(@Nullable Specification<T> spec) {
        if (spec != null) {
            specifications.add(spec);
        }
        return this;
    }

    /**
     * Создает объединенную спецификацию на основе всех добавленных спецификаций.
     *
     * @return Объединенная спецификация.
     */
    public Specification<T> build() {
        if (specifications.isEmpty()) {
            return Specification.where(null);
        }

        Specification<T> result = Specification.where(specifications.get(0));
        for (int i = 1; i < specifications.size(); i++) {
            result = result.and(specifications.get(i));
        }
        return result;
    }

    /**
     * Возвращает неизменяемый список спецификаций, добавленных в билдер.
     *
     * @return Список спецификаций.
     */
    public List<Specification<T>> getSpecifications() {
        return Collections.unmodifiableList(specifications);
    }
}
```
```java
import jakarta.persistence.criteria.CriteriaBuilder;
import jakarta.persistence.criteria.CriteriaQuery;
import jakarta.persistence.criteria.Predicate;
import jakarta.persistence.criteria.Root;
import org.springframework.data.jpa.domain.Specification;
import lombok.NonNull;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

/**
 * Спецификация фильтрации для сущности PaymentAnalyticEntity.
 *
 * @author Владислав Турукин
 */
public class PFLoanFundsRqRegistryPaymentAnalyticFilterSpecification implements Specification<PaymentAnalyticEntity> {

    private static final String INN_MAYBE_REGEX = "\\d{1,12}";
    @NonNull
    private final PFLoanFundsRqRegistryFilterDto filter;

    /**
     * Конструктор с параметрами фильтрации.
     *
     * @param filter DTO для фильтрации.
     */
    public PFLoanFundsRqRegistryPaymentAnalyticFilterSpecification(@NonNull PFLoanFundsRqRegistryFilterDto filter) {
        this.filter = filter;
    }

    @Override
    public Predicate toPredicate(@NonNull Root<PaymentAnalyticEntity> root,
                                 @NonNull CriteriaQuery<?> query,
                                 @NonNull CriteriaBuilder cb) {
        List<Predicate> predicates = new ArrayList<>();

        /**
         * Фильтрация по статусам
         */
        if (filter.getLoanFundsRequestStatuses() != null && !filter.getLoanFundsRequestStatuses().isEmpty()) {
            Set<PaymentAnalyticStatus> statuses = filter.getLoanFundsRequestStatuses().stream()
                    .map(PaymentAnalyticStatus::of)
                    .collect(Collectors.toUnmodifiableSet());
            predicates.add(root.get(PaymentAnalyticEntity_.status).in(statuses));
        }

        /**
         * Фильтрация по номеру
         */
        if (filter.getNumber() != null && !filter.getNumber().isEmpty()) {
            Expression<String> numberAsString = cb.function("cast", String.class, root.get(PaymentAnalyticEntity_.number));
            predicates.add(cb.like(numberAsString, filter.getNumber() + "%"));
        }

        /**
         * Фильтрация по дате
         */
        if (filter.getLoanFundsRequestDate() != null) {
            if (filter.getLoanFundsRequestDate().getFrom() != null) {
                predicates.add(cb.greaterThanOrEqualTo(root.get(PaymentAnalyticEntity_.date), filter.getLoanFundsRequestDate().getFrom()));
            }
            if (filter.getLoanFundsRequestDate().getTo() != null) {
                predicates.add(cb.lessThanOrEqualTo(root.get(PaymentAnalyticEntity_.date), filter.getLoanFundsRequestDate().getTo()));
            }
        }

        /**
         * Фильтрация по счетам плательщиков
         */
        if (filter.getPayerAccounts() != null && !filter.getPayerAccounts().isEmpty()) {
            predicates.add(root.get(PaymentAnalyticEntity_.payerAccount).in(filter.getPayerAccounts()));
        }

        /**
         * Фильтрация по счетам получателей
         */
        if (filter.getRecipientAccounts() != null && !filter.getRecipientAccounts().isEmpty()) {
            predicates.add(root.get(PaymentAnalyticEntity_.recipientAccount).in(filter.getRecipientAccounts()));
        }

        /**
         * Фильтрация по сумме
         */
        if (filter.getAmountFrom() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get(PaymentAnalyticEntity_.amount), filter.getAmountFrom()));
        }
        if (filter.getAmountTo() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get(PaymentAnalyticEntity_.amount), filter.getAmountTo()));
        }

        /**
         * Фильтрация по цели платежа
         */
        if (filter.getPaymentPurposeSearch() != null && !filter.getPaymentPurposeSearch().isEmpty()) {
            predicates.add(cb.like(cb.lower(root.get(PaymentAnalyticEntity_.paymentPurpose)),
                    filter.getPaymentPurposeSearch().toLowerCase() + "%"));
        }

        /**
         * Фильтрация по кредитным соглашениям
         */
        if (filter.getCreditAgreementIds() != null && !filter.getCreditAgreementIds().isEmpty()) {
            Set<UUID> creditAgreementIds = filter.getCreditAgreementIds().stream()
                    .map(UUID::fromString)
                    .collect(Collectors.toUnmodifiableSet());
            predicates.add(root.get(PaymentAnalyticEntity_.creditAgreementNumber).in(creditAgreementIds));
        }

        /**
         * Фильтрация по дате выпуска транша
         */
        if (filter.getTrancheIssueDate() != null) {
            if (filter.getTrancheIssueDate().getFrom() != null) {
                predicates.add(cb.greaterThanOrEqualTo(root.get(PaymentAnalyticEntity_.trancheIssueDate), filter.getTrancheIssueDate().getFrom()));
            }
            if (filter.getTrancheIssueDate().getTo() != null) {
                predicates.add(cb.lessThanOrEqualTo(root.get(PaymentAnalyticEntity_.trancheIssueDate), filter.getTrancheIssueDate().getTo()));
            }
        }

        /**
         * Фильтрация по получателю или ИНН
         */
        if (filter.getRecipientNameOrInn() != null && !filter.getRecipientNameOrInn().isEmpty()) {
            String val = filter.getRecipientNameOrInn();
            Predicate namePredicate = cb.like(cb.lower(root.get(PaymentAnalyticEntity_.recipientName)), val.toLowerCase() + "%");
            if (val.matches(INN_MAYBE_REGEX)) {
                Predicate innPredicate = cb.like(root.get(PaymentAnalyticEntity_.recipientInn), val + "%");
                predicates.add(cb.or(namePredicate, innPredicate));
            } else {
                predicates.add(namePredicate);
            }
        }

        /**
         * Фильтрация по типу фонда
         */
        if (filter.getFundsType() != null && !filter.getFundsType().isEmpty()) {
            predicates.add(cb.equal(root.get(PaymentAnalyticEntity_.fundsType), FundsType.of(filter.getFundsType())));
        }

        /**
         * Фильтрация по типу платежа
         */
        if (filter.getPaymentType() != null && !filter.getPaymentType().isEmpty()) {
            predicates.add(cb.equal(root.get(PaymentAnalyticEntity_.paymentType), PaymentType.of(filter.getPaymentType())));
        }

        return cb.and(predicates.toArray(new Predicate[0]));
    }
}
```
```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import lombok.NonNull;
import jakarta.persistence.EntityManager;
import jakarta.persistence.criteria.*;

import java.util.*;
import java.util.stream.Collectors;

/**
 * Интерфейс для сервиса группировки и подсчета элементов.
 *
 * @param <E> Тип сущности, по которой производится подсчет.
 */
public interface GroupCountService<E> {

    /**
     * Подсчитывает количество элементов в группах на основе заданного типа подсчета.
     *
     * @param countEnum      Тип подсчета из {@link RegistryCountEnum}.
     * @param specification  Спецификация для фильтрации.
     * @param <E>            Тип значения группы.
     * @return Карта, где ключ - {@link RegistryCountEnum}, значение - список пар (значение группы, количество).
     */
    <E> Map<RegistryCountEnum, List<Pair<E, Long>>> countByGroup(@NonNull RegistryCountEnum countEnum,
                                                                 @NonNull Specification<PaymentAnalyticEntity> specification);
}


/**
 * Реестр для определения путей группировки на основе RegistryCountEnum.
 */
public class CountGroupMappingRegistry {

  private static final Map<RegistryCountEnum, BiFunction<Root<PaymentAnalyticEntity>, CriteriaBuilder, List<Expression<?>>>> GROUP_BY_MAPPING = new EnumMap<>(RegistryCountEnum.class);

  static {
    // Группировка по статусу
    GROUP_BY_MAPPING.put(RegistryCountEnum.STATUS_COUNT, (root, cb) ->
            List.of(root.get(PaymentAnalyticEntity_.status)));

    // Группировка по получателю
    GROUP_BY_MAPPING.put(RegistryCountEnum.RECIPIENT_COUNT, (root, cb) ->
            List.of(root.get(PaymentAnalyticEntity_.recipientName)));

    // Группировка по типу платежа
    GROUP_BY_MAPPING.put(RegistryCountEnum.PAYMENT_TYPE_COUNT, (root, cb) ->
            List.of(root.get(PaymentAnalyticEntity_.paymentType)));

    // Дополнительные группировки можно добавлять здесь.
  }

  /**
   * Возвращает пути для группировки на основе заданного типа подсчета.
   *
   * @param countEnum Тип подсчета из {@link RegistryCountEnum}.
   * @param root      Корневой объект сущности PaymentAnalyticEntity.
   * @param cb        Объект CriteriaBuilder для построения выражений.
   * @return Список путей ({@link Expression}) для группировки.
   * @throws IllegalArgumentException Если не найдено сопоставление для заданного типа подсчета.
   */
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

@Service
public class GroupCountServiceImpl implements GroupCountService<PaymentAnalyticEntity> {

  private final EntityManager entityManager;

  public GroupCountServiceImpl(EntityManager entityManager) {
    this.entityManager = entityManager;
  }

  @Override
  @Transactional(readOnly = true)
  public <E> Map<RegistryCountEnum, List<Pair<E, Long>>> countByGroup(
          @NonNull RegistryCountEnum countEnum,
          @NonNull Specification<PaymentAnalyticEntity> specification) {

    CriteriaBuilder cb = entityManager.getCriteriaBuilder();
    CriteriaQuery<Tuple> query = cb.createTupleQuery();
    Root<PaymentAnalyticEntity> root = query.from(PaymentAnalyticEntity.class);

    // Получаем поля для группировки
    List<Expression<?>> groupByPaths = CountGroupMappingRegistry.getGroupByPaths(countEnum, root, cb);

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
```java

```
```java
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
import org.springframework.stereotype.Repository;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.lang.Nullable;
import lombok.NonNull;

import java.util.UUID;

/**
 * Репозиторий для работы с сущностью PaymentAnalyticEntity.
 *
 * @author Владислав Турукин
 */
@Repository
public interface PaymentAnalyticRepository extends
        PagingAndSortingRepository<PaymentAnalyticEntity, UUID>,
        JpaSpecificationExecutor<PaymentAnalyticEntity> {

  /**
   * Находит все записи с применением фильтрации и сортировки, подгружая связанные сущности.
   *
   * @param spec     Спецификация фильтрации.
   * @param pageable Объект пагинации.
   * @return Страница результатов с загруженными связанными объектами.
   */
  @EntityGraph(attributePaths = {
          "paymentObjects",
          "paymentObjects.paymentObjectSsr",
          "paymentObjects.paymentObjectSsr.paymentSsrArticles"
  })
  Page<PaymentAnalyticEntity> findAll(@Nullable Specification<PaymentAnalyticEntity> spec, @NonNull Pageable pageable);

  /**
   * Подсчитывает количество записей, удовлетворяющих спецификации.
   *
   * @param spec Спецификация фильтрации.
   * @return Количество записей.
   */
  long count(@Nullable Specification<PaymentAnalyticEntity> spec);
}

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
lombok.NonNull;
import org.springframework.lang.Nullable;

import java.util.Map;
import java.util.List;

/**
 * Сервис для работы с PaymentAnalyticEntity.
 *
 * @author Владислав Турукин
 */
@Service
public class PaymentAnalyticService {

  private final PaymentAnalyticRepository repository;
  private final GroupCountService groupCountService;

  public PaymentAnalyticService(PaymentAnalyticRepository repository, GroupCountService groupCountService) {
    this.repository = repository;
    this.groupCountService = groupCountService;
  }

  /**
   * Получает отфильтрованные и отсортированные записи.
   *
   * @param specification Спецификация фильтрации и сортировки.
   * @param pageable       Объект пагинации.
   * @return Страница результатов.
   */
  @Transactional(readOnly = true)
  public Page<PaymentAnalyticEntity> getFilteredAndSortedData(
          @Nullable Specification<PaymentAnalyticEntity> specification,
          @NonNull Pageable pageable) {

    return repository.findAll(specification, pageable);
  }

  /**
   * Подсчитывает количество элементов в группах на основе фильтрации.
   *
   * @param countEnum      Тип группировки (например, по статусу, получателю).
   * @param specification  Спецификация фильтрации.
   * @return Карта групп и их количества.
   */
  @Transactional(readOnly = true)
  public <E> Map<RegistryCountEnum, List<Pair<E, Long>>> getCountByGroups(
          @NonNull RegistryCountEnum countEnum,
          @Nullable Specification<PaymentAnalyticEntity> specification) {

    return groupCountService.countByGroup(countEnum, specification);
  }
}

```
