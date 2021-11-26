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
3. Result. Expected result of parsing. It consists of multiple steps and metadata.
4. Direction item. Every step consists of multiple direction items. There're four types:
    * `text` has `value` which contains text used in step.
    * `ingredient` has `name`, `quantity` and `units`.
    * `cookware` has `name` and `quantity`.
    * `timer` has `name`, `quantity` and `units`.
5. Metadata is in key/value form.

Example usage in https://github.com/cooklang/CookInSwift.
