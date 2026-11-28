# Canonical tests for parsers

## Datastructure

Every tests looks like this:

```
testBasicDirection: ‹ 1
    source: |               ‹ 2
      Add a bit of chilli
    result:                      ‹ 3
      steps:
        -
          - type: text                   ‹ 4
            value: "Add a bit of chilli"
      metadata:                             ‹ 5
        "cooking time": 30 mins
```

Key components:

1. Test name. It gives an idea what feature this test is testing and can be useful for debugging.
2. Source. Recipe text file as it'd provided by a user.
3. Result. Expected result of parsing. It consists of multiple steps, notes and metadata.
4. Direction item. Every step consists of multiple direction items. There're four types:
    * `text` has `value` which contains text used in step.
    * `ingredient` has `name`, `quantity` and `units`, and `optional: true` when marked with `?` (`optional` defaults to `false` and is omitted when false).
    * `cookware` has `name` and `quantity`, and `optional` like ingredients.
    * `timer` has `name`, `quantity` and `units`.

    A `quantity` is a number (`2`, `0.5`), a string for a text quantity (`"a pinch"`), or, for a [range](../README.md#ranges), a mapping with `start` and `end`:

    ```yaml
    quantity:
      start: 2
      end: 3
    ```
5. Metadata is in key/value form.
6. Notes (optional, a test without `notes` expects none) are listed in order, each as a list of text runs. A run has `type: text` and `value`, plus `strong: true`, `emphasis: true` and `link` (the destination) when the run has that formatting; they are omitted otherwise:

    ```yaml
    notes:
      -
        - type: text
          value: "Make ahead:"
          strong: true
        - type: text
          value: " the dough keeps "
        - type: text
          value: "shaping guide"
          link: "https://example.com/shaping"
    ```

Example usage in https://github.com/cooklang/CookInSwift.
