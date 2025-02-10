# Ticket: Implement Holiday Schedule for Symbols

## Business Value

Currently, brokers have no convenient way to define holidays for trading symbols.
They must manually specify the symbols intervals before and after a holiday,
leading to inefficiencies and potential errors.
Implementing a dedicated holiday scheduling feature will:
- Reduce manual intervention in symbols intervals;
- Prevent trading on designated holidays;
- Ensure consistency and accuracy in trading schedules;
- Improve broker operational efficiency;

## Description

We need to extend the existing trading schedule management functionality by introducing a Holiday Schedule feature. 
Brokers should be able to define full-day holidays for each symbol, ensuring that trading is automatically disabled 
for the selected dates.

### Requirements:
- Introduce a new Holiday Schedule entity (`Holiday`) associated with symbols;
- Modify API for creating, updating, and deleting holidays;
- Modify existing admin application logic for API to prevent trading on configured holidays;
  - Return the symbol intervals considering the weekend;
- Emit system events when a holiday schedule is modified;

Holiday entity:

| Field        | Type     | Constraints | Description               |
|--------------|----------|-------------|---------------------------|
| `holiday_id` | `Long`   | `PK`        | Holiday id                |
| `symbol_id`  | `Long`   | `FK`        | A financial instrument Id |
| `date_start` | `Date`   | `NotNull`   | Holiday date start        |
| `date_end`   | `Date`   | `NotNull`   | Holiday date end          |
| `reason`     | `String` |             | Holiday description       |



## Use Cases
### 1. Broker sets a holiday for a symbol
#### Description: 
The broker configures a specific date when the symbol should not be tradable.
#### Business logic:
- Broker wants to disallow trading a symbol on a certain days;
- The dates of the holiday is specified without timezone;
- Once a holiday is set, traders will not be able to place orders for that symbol on that day;
#### Technical details:
1. The request is sent in an asynchronous `CUDHolidayRequest` message;
2. The holiday date is stored in the database with the date type (ISO 8601 YYYY-MM-DD);
3. After successful holiday creation the system sends asynchronous event `CUDHolidayResponse`;
#### Validations:
1. The start date must not be earlier than the current date (holidays cannot be set in the past);
2. More than one holiday per symbol cannot be placed on the same day;
3. The dates must be in the format `YYYY-MM-DD`;
4. For `CREATE` operation `symbol_id`, `date_start` and `date_end` is required;
5. If start_date == end_date, it is a single holiday;
6. Holidays do not overlap (no two holidays can overlap the same dates by symbol);
7. start_date <= end_date (cannot specify the end before the start);

#### Request body example:
```json
{
  "symbol_id": 1001,
  "operation": "CREATE",
  "date_start": "2025-01-01",
  "date_end": "2025-01-03",
  "reason": "New Year's Days"
}
```
#### Response body example:
```json
{
  "symbol_id": 1001,
  "holiday_id": 12345,
  "operation": "CREATE",
  "date_start": "2025-01-01",
  "date_end": "2025-01-03",
  "reason": "New Year's Days"
}
```

### 2. Broker updates holiday
#### Description:
The broker modifies an existing holiday (e.g., changes the dates).
#### Business logic:
- If the holiday is rescheduled, the broker should be able to change the dates;
#### Technical details:
1. The update is performed via an asynchronous `CUDHolidayRequest` message;
2. After a successful update, the system sends a `CUDHolidayResponse` message;
#### Validations:
1. A holiday must exist;
2. The start date must not be earlier than the current date;
3. For `UPDATE` operation `symbol_id`, `holiday_id`, `date_start` and `date_end` is required;
4. If start_date == end_date, it is a single holiday;
5. Holidays do not overlap (no two holidays can overlap the same dates by symbol);
6. start_date <= end_date (cannot specify the end before the start);
#### Request body example:
```json
{
  "symbol_id": 1001,
  "holiday_id": 12345,
  "operation": "UPDATE",
  "date_start": "2025-01-02",
  "date_end": "2025-01-03",
  "reason": "New Year's Days"
}
```
#### Response body example:
```json
{
  "symbol_id": 1001,
  "holiday_id": 12345,
  "operation": "UPDATE",
  "date_start": "2025-01-02",
  "date_end": "2025-01-03",
  "reason": "New Year's Days"
}
```

### 3. Broker removes holiday
#### Description:
Broker removes a holiday.
#### Business logic:
- If the holiday is no longer needed, it should be removed;
#### Technical details:
- The deletion is performed via an asynchronous `CUDHolidayRequest` message;
- After deletion, the system sends the `CUDHolidayResponse` message;
#### Validations:
- Only an existing holiday can be deleted;
- For `DELETE` operation `holiday_id` is required;
- Optional: repeated deletion must not cause an error if the holiday has already been deleted (idempotency);
#### Request body example:
```json
{
  "holiday_id": 12345,
  "operation": "DELETE"
}
```
#### Response body example:
```json
{
  "symbol_id": 1001,
  "holiday_id": 12345,
  "operation": "DELETE",
  "date_start": "2025-01-02",
  "date_end": "2025-01-03",
  "reason": "New Year's Day"
}
```
### 4. Trader attempts to place an order on a holiday
#### Description:
Trader attempts to place an order on a holiday
#### Business logic:
- Trading the symbol on a holiday is not allowed;
- An attempt to place an order must be rejected with a corresponding error;
#### Technical details:
- When a trader tries to send an order placement request, the system checks the symbol intervals that correspond to a given holiday;
- If the symbol is on a holiday, an error is returned;

### 5. Broker requests a list of all holidays or for a specified symbol
#### Description:
Broker requests a list of all holidays or for a specified symbol
#### Business logic:
- Broker may request all designated holidays for a symbol;
#### Technical details:
- The request is sent via an asynchronous `GetHolidayListRequest` message;
- In response the system returns a list of all holidays in `GetHolidayListResponse`;
#### Request body example:
```json
{
  "symbol_id": 1001
}
```
#### Response body example:
```json
{
  "holidays": [
    {
      "symbol_id": 1001,
      "holiday_id": 12345,
      "date_start": "2025-01-02",
      "date_end": "2025-01-03",
      "reason": "New Year's Day"
    }
  ]
}
```

### 6. The system publishes the HolidayChangedEvent when holidays are changed
#### Description:
When a holiday is added, updated, or removed, `HolidayChangedEvent` is triggered.
#### Business logic:
- Any change in holidays must be recorded and sent to all services concerned;
#### Technical details:
- The system generates a `HolidayChangedEvent` with information about the changes;
#### Event format:
```json
{
  "symbol_id": 1001,
  "holiday_id": 12345,
  "operation": "DELETE",
  "date_start": "2025-01-02",
  "date_end": "2025-01-03",
  "reason": "New Year's Days"
}
```


## API Protocol

| Message                  | Body                                                                                                                                                                                                                              | Description                                                  |
|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| `GetHolidayListRequest`  | optional `symbol_id`: `int64`                                                                                                                                                                                                     | Get list of existing holidays. symbol_id is optional filter. |
| `GetHolidayListResponse` | required `holidays`: `Holiday[]`                                                                                                                                                                                                  |                                                              |
| `CUDHolidayRequest`      | optional `symbol_id`: `int64`<br/> optional `holiday_id`: `int64`<br/> required `operation`: `enum (CREATE, UPDATE, DELETE)`<br/> optional `date_start`: `date`<br/> optional `date_end`: `date`<br/> optional `reason`: `string` | Create/Update/Delete holiday                                 |
| `CUDHolidayResponse`     | required `symbol_id`: `int64`<br/> required `holiday_id`: `int64`<br/> required `operation`: `enum (CREATE, UPDATE, DELETE)`<br/> required `date_start`: `date`<br/> required `date_end`: `date`<br/> optional `reason`: `string` | Modification is done successfully                            |
| `HolidayChangedEvent`    | required `symbol_id`: `int64`<br/> required `holiday_id`: `int64`<br/> required `operation`: `enum (CREATE, UPDATE, DELETE)`<br/> required `date_start`: `date`<br/> required `date_end`: `date`<br/> optional `reason`: `string` | Server event on holiday modification                         |


## Test cases

| Test Case                                | Role                          | Action                           | Expected Result                                                                                  |
|------------------------------------------|-------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------|
| Create a holiday                         | Broker                        | Create message CUDHolidayRequest | Holiday is created and returned in `GetHolidayListRequest`                                       |
| Update a holiday                         | Broker                        | Update message CUDHolidayRequest | All holiday fields provided in `CUDHolidayRequest` saved and returned in `GetHolidayListRequest` |
| Delete a holiday                         | Broker                        | Delete message CUDHolidayRequest | Holiday deleted and no longer returned in the request `GetHolidayListRequest`                    |
| Prevent trading on a holiday             | Trader (or mb Trading Server) | Attempt to place an order        | Order is rejected with an error message                                                          |
| Holiday appears in `HolidayChangedEvent` | Broker Admin backend          | Create/update/delete a holiday   | `HolidayChangedEvent` is published with correct details                                          |