# CMS Validation

## Requirements

- Prevent cms user from entering invalid data.
- Allow developer to use common and customisable validation in cms config on field level and form level.
- Display error message, which can be defined in config, for invalid data.

## Common Use Cases

- Non empty text input.
- Text input with max length.
- Text input with certain pattern, e.g. credit card number.
- Number or datetime input value within range.
- One field cannot be empty if another field is not empty.

## Overview

- Validation is a process that evaluate data with an expression, the result is either success or failure.
- Validation can be defined in record edit page and new page in cms config.
- Validation happens after use click save button and before all server api calls are fired.
- If any validation result is failure, error message should be displayed and no server api calls should be fired.
- Error message of field validation appears next / below the field, while that of form validation appears below the form.
- Field validation is executed before form validation.

## Configuration

- One validation item is a dictionary which contains the validation configuration (depends on where it is placed).
  - Each validation optionally contains a `message` field, which would be the error message if that validation item cannot pass.
  - Otherwise, a default error message would be shown.

```
new:
  # page validation
  validations:
    - expression: xxx
    - expression: xxx
  fields:
    - name: xxx
      type: xxx
      # field validation
      validations:
        - regex: /^xxx$/
          message:
```

## Expression

- `expression` is the base type of validation, supported by all validation.
- An expression is a string, which can be evaluated to give boolean value.
- All other types of validation are predifined expression, i.e. they are all compiled to expression at runtime.
- Expression evaluation would be based on this library https://github.com/joewalnes/filtrex, while CMS would provide extra predefined functions for common use case.

### Expression context: `value`

Developer uses the variable `value` in the expression to get the value from the field or form.

- Field validation
  - `value`: value of the field
  - e.g. `value in ('abc', 'def')`
- Form validation
  - `value`: value of the form
  - `get(value: formValue, fieldName: string)`
  - e.g. `get(value, "full_name") != get(value, "nick_name")`

Type of `value` for each field type

- String: string
- Number: number
- Datetime: datetime
- JSON: object | array
- Location: object<{ lat: number, lng: number }>
- Reference: object
- References: array<object>
- EmbeddedReference: object
- EmbeddedReferences: array<object>
- Asset: object<{ contentType: string, size: number }>

`get` can be used to get children value in object(-like) and array(-like) value.

### Functions

- Passing wrong data type to the function would throw error in validation.

#### Predefined functions

- String
  - `length(value: string)`
  - `upper(value: string)`
  - `lower(value: string)`
  - `substring(value: string, from: number, to: number)`
  - `regex(value: string, regex: string)`
  - `match_pattern(value: string, pattern: string)`
    - e.g. `email`, `credit_card`, `url`
- Number
- Datetime
  - `datetime(str: string)`
  - `timestamp(data: datetime | string)`
    - e.g. `timestamp(value) > timestamp('2018-01-01')`
  - `get_year(data: datetime)`
  - `get_month(data: datetime)`
  - `get_week_of_year(data: datetime)`
  - `get_day_of_month(data: datetime)`
  - `get_day_of_year(data: datetime)`
  - `get_day_of_week(data: datetime)`
  - `get_hour(data: datetime)`
  - `get_minute(data: datetime)`
  - `get_second(data: datetime)`
  - TODO: Provide a complete set of date time functions, e.g. exposing `moment` functions
- JSON
  - `get(data: json, key: string | number)`
    - return the value at key
    - e.g. `get(get(value, "a"), "b")`, `{"a": {"b": 1}}` => `1`
    - e.g. `get(get(value, "a"), 0)`, `{"a": [1]}` => `1`
  - `has_key(data: json, key: string | number)`
    - return if the key exist
  - `typeof(data: json)`
    - return `object`, `array`, `string`, `number`, `null`
  - `length(data: json)`
    - return length of json array
    - throw error if provided data is not json array
- Location
- References
  - `length(value: reference[])`
- EmbeddedReference
  - `get(data: embeddedReference, key: string)`
- EmbeddedReferences
  - `length(value: embeddedReference[])`
  - `get(data: embeddedReference[], index: number)`
- Asset

## Predefined validation for field validation

### `required`

types: `String`, `Datetime`, `References`, `EmbeddedReferences`, `Asset`

```
validations:
- required: true
# String / References / EmbeddedReferences equivalent to
- expression: length() > 0
# Datetime / Asset equivalent to
- expression: value != null
```

Note that, `required: false` is no-op.

### `regex`

types: `String`

```
validations:
- regex: /xxx/
# equivalent to
- expression: value ~= /xxx/
```

### `pattern`

Call `match_pattern`, e.g. `email`, `credit_card`, `url`

types: `String`

```
validations:
- pattern: email
```

### `length`

types: `String`, `References`, `EmbeddedReferences`

- `min`, `max`, can set one of them or both of them
- `inclusive`: boolean, optional, default true

```
validations:
- length:
    min: 3
    max: 5
    inclusive: true
# String / References / EmbeddedReferences equivalent to
- expression: length() >= 3 and length() <= 5
```

### Comparisons

types: `String`

- `contains`
- `not_contains`

types: `String`, `Number`, `Datetime`

- `equal_to`
- `not_equal_to`

types: `Number`, `Datetime`

- `less_than`
- `less_than_or_equal_to`
- `greater_than`
- `greater_than_or_equal_to`
- `range`
  - `min`, `max`, can set one of them or both of them
  - `inclusive`: boolean, optional, default true

```
validations:
- greater_than: 2018-01-01
- range:
    min: 2000-01-01
    max: 2001-01-01
```

### Null value handling

Cases where `null` value appears:

- value cleared for `Datetime`, `Reference`, `EmbeddedReference`, `Asset` field
- value was originally `null` and not updated when save

They may produce error when calling predefined functions, since `null` is a different data type from its original data type of the field type.

Null value can be eliminated by adding extra checking in the expression, but this checking will be duplicated in most expression and may complicate the expression, which makes it hard to maintain.

```yaml
# bad example
# Error is thrown if value == null, because length requires a string or array
- expression: length(value) > 10

# example
- expression: value == null or length(value) > 10
```

`when` config is introduced to simplify the expression.

- accepts an string expression which evaluates to give boolean value, like `expression`.
- the effective boolean expression would be: `not (when) or (expression)`
- `when` is optional, default `when` expression: `true`
- if `when` expression pass, `expression` will evaluate the value.
- if `when` expression fail, the validation will pass.

```yaml
# example with 'when'
- when: value != null
  expression: length(value) > 10
  # equivalent to
  expression: not (value != null) or (length(value) > 10)
```

#### Field validation

All validations, except `required`, require non null value to work.

Thus we provide a default `when` config, `value != null`, for all validations except `required`.

In other word, **CMS developer does NOT need to check null value themselves in field validation.**

```yaml
# example: field not required
- type: date_time
  validations:
    - expression: get_year(value) >= 2000
    - expression: get_month(value) >= 6 and get_month(value) <= 8

# example: field required
- type: date_time
  validations:
    - required: true  # reject null value
    - expression: get_year(value) >= 2000
    - expression: get_month(value) >= 6 and get_month(value) <= 8

# example: field required (effectively same as above)
- type: date_time
  validations:
    - expression: get_year(value) >= 2000
    - expression: get_month(value) >= 6 and get_month(value) <= 8
    - required: true  # reject null value

# BAD example:
- type: date_time
  validations:
    - expression: value != null  # this will always pass
```

#### Form validation

No default value for `when` config in form validation.

```yaml
# example
- when: get(value, "deadline") != null and get(value, "submitted_at") != null
  expression: timestamp(get(value, "deadline")) > timestamp(get(value, "submitted_at"))

# example (expression only version)
- expression: get(value, "deadline") == null or get(value, "submitted_at") == null or timestamp(get(value, "deadline")) > timestamp(get(value, "submitted_at"))
```

#### Other use case

Checking `null` value is only one use case of `when` config, CMS developer can organize the validation config by combining `when` and `expression` config, or other predefined validation config, in their own fashion.

However, the original `when` config will be overwritten, thus the null value checking must be written explicitly.

```yaml
# example
- type: string
  validations:
    - when: value != null and regex(value, "^admin")
      length:
        max: 15
    - when: value != null and not regex(value, "^admin")
      length:
        max: 10
```
