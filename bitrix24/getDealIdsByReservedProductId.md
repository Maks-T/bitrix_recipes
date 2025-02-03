# Сборник сниппетов кода для Bitrix 24

## Получение ID сделок по ID зарезервированного товара

Этот сниппет позволяет получить список ID сделок, связанных с зарезервированным товаром по его ID. Используется для работы с корзиной и резервированием товаров в Bitrix.

### Описание

Функция `getDealIdsByReservedProductId` принимает на вход ID товара и возвращает массив данных, содержащий информацию о сделках, связанных с этим товаром. В частности, возвращаются ID сделок, ID заказов, количество товара и дата окончания резервирования.

### Код

```php
private function getDealIdsByReservedProductId($productId)
{
    return \Bitrix\Sale\Basket::getList([
        'select' => [
            'ID',
            'ORDER_ID',
            'QUANTITY',
            'DEAL_ID' => 'ENTITY_BINDING.OWNER_ID',
            'BASKET_RESERVED.DATE_RESERVE_END'
        ],
        'filter' => [
            'PRODUCT_ID' => $productId,
            '<=BASKET_RESERVED.DATE_RESERVE_END' => array(false, ConvertTimeStamp(false, "FULL"))
        ],
        'runtime' => [
            new \Bitrix\Main\Entity\ReferenceField(
                'BASKET_RESERVED',
                '\Bitrix\Sale\Reservation\Internals\BasketReservationTable',
                [
                    '=this.ID' => 'ref.BASKET_ID',
                ],
                ['join_type' => 'INNER']
            ),
            new \Bitrix\Main\Entity\ReferenceField(
                'ENTITY_BINDING',
                'Bitrix\Crm\Binding\OrderEntityTable',
                [
                    '=ref.ORDER_ID' => 'this.ORDER_ID',
                    '=ref.OWNER_TYPE_ID' => new \Bitrix\Main\DB\SqlExpression("?i", \CCrmOwnerType::Deal),
                ],
                ['join_type' => 'INNER']
            ),
        ]
    ])->fetchAll();
}

```

### Описание runtime-запросов и структур таблиц

В коде используются два `ReferenceField` в `runtime`-разделе запроса, которые позволяют связать таблицу корзины (`Bitrix\Sale\Basket`) с таблицами резервирования товаров и привязки к сделкам.

#### 1. `BASKET_RESERVED`
Связывает корзину с таблицей резервирования товаров (`Bitrix\Sale\Reservation\Internals\BasketReservationTable`):
- **`BASKET_RESERVED.DATE_RESERVE_END`** — дата окончания резервирования товара.
- **`BASKET_RESERVED.BASKET_ID`** связывается с `ID` в таблице `Basket`, что позволяет получать данные о резервировании для конкретного товара.

#### 2. `ENTITY_BINDING`
Связывает заказы с таблицей `Bitrix\Crm\Binding\OrderEntityTable`, которая содержит связи заказов с CRM-сущностями (например, сделками).
- **`ENTITY_BINDING.OWNER_ID`** — ID сделки (`DEAL_ID`).
- **`ENTITY_BINDING.OWNER_TYPE_ID`** фильтруется значением `\CCrmOwnerType::Deal`, что гарантирует выборку только сделок, связанных с заказами.
- **`ENTITY_BINDING.ORDER_ID`** связывается с `ORDER_ID` из таблицы `Basket`, что позволяет получить ID заказа, связанного с данной сделкой.

### Типы объединений (join_type)

В коде используется `['join_type' => 'INNER']`, что означает, что в результат попадут только записи, у которых есть соответствие в обеих таблицах. Этот тип объединения позволяет исключить элементы, для которых не найдено связанной информации.

Другие возможные типы объединений:
- **`LEFT`** — выборка всех записей из основной таблицы и соответствующих записей из связанной таблицы (если их нет, будут NULL-значения).
- **`RIGHT`** — аналогично `LEFT`, но в обратном направлении.
- **`OUTER`** — объединяет все записи из обеих таблиц, заполняя отсутствующие значения NULL.

