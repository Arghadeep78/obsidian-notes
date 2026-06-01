## DOM Nodes vs Elements

- **DOM Node:** The base class for everything in the DOM tree (Elements, Text, Comments, Document).
    
- **Element Node:** A specific type of node represented by HTML tags (e.g., `<div>`, `<p>`).
    

```
Document
└── div (Element Node)
    ├── "Hello" (Text Node)
    ├── (Comment Node)
    └── span (Element Node)
```

- **element.childNodes:** Returns a NodeList containing all node types (text, comments, elements).
    
- **element.children:** Returns an HTMLCollection containing only element nodes.
    

## DOM Query Methods

|**Method**|**Returns**|**Selection Criteria**|
|---|---|---|
|`getElementById()`|Single Element|By ID|
|`querySelector()`|Single Element|By CSS Selector (First match)|
|`querySelectorAll()`|NodeList|By CSS Selector (All matches)|
|`getElementsByClassName()`|HTMLCollection|By Class Name|
|`getElementsByTagName()`|HTMLCollection|By Tag Name|

> **Interview Shortcut:** Singular methods (`getElementById`, `querySelector`) return a single Element. Plural methods (`getElementsBy...`, `querySelectorAll`) return a collection. `querySelector()` is preferred in modern development for its CSS selector flexibility.

## NodeList vs HTMLCollection

| **Feature**                     | **NodeList**                      | **HTMLCollection**                |
| ------------------------------- | --------------------------------- | --------------------------------- |
| Indexed Access (`[0]`)          | ✅ Yes                             | ✅ Yes                             |
| `.length`                       | ✅ Yes                             | ✅ Yes                             |
| `.forEach()`                    | ✅ Yes                             | ❌ No                              |
| `for...of Loop                  | ✅ Yes                             | ✅ Yes                             |
| Array Methods (`map`, `filter`) | ❌ No (Convert via `Array.from()`) | ❌ No (Convert via `Array.from()`) |
| **Live / Static**               | Usually Static                    | Always Live                       |

## innerHTML vs outerHTML

- **innerHTML:** Gets/sets the HTML markup inside the element.
    
- **outerHTML:** Gets/sets the HTML of the element itself plus its content. Setting it replaces the element entirely.
    

## Functions: Declarations vs Expressions

|**Feature**|**Function Declaration (function abc() {})**|**Function Expression (const abc = function() {})**|
|---|---|---|
|**Hoisting**|✅ Full function is hoisted|❌ Only variable declaration is hoisted|
|**Callable Before Line**|✅ Yes|❌ No (`ReferenceError`)|
|**Reassignment**|✅ Yes (can be redeclared)|❌ No (if declared with `const`)|

## Array Methods Cheat Sheet

| **Method**      | **Returns**           | **Purpose**                                        | **Mutates Original?** |
| --------------- | --------------------- | -------------------------------------------------- | --------------------- |
| **`forEach()`** | `undefined`           | Iterates over elements (cannot break early).       | ❌ No                  |
| **`map()`**     | New Array             | Transforms each element via a callback.            | ❌ No                  |
| **`filter()`**  | New Array             | Selects elements that pass a condition.            | ❌ No                  |
| **`find()`**    | Element / `undefined` | Returns the first element that passes a condition. | ❌ No                  |
| **`some()`**    | Boolean               | Checks if at least one element passes a condition. | ❌ No                  |
| **`every()`**   | Boolean               | Checks if all elements pass a condition.           | ❌ No                  |
| **`reduce()`**  | Single Value          | Aggregates array elements into a single output.    | ❌ No                  |
| **`sort()`**    | Sorted Array          | Sorts elements in place.                           | ✅ Yes                 |

### Advanced `map()` Parameters

JavaScript

```
arr.map(function(element, index, array) { ... }, thisArg);
```

- **Callback arguments:** Current element, current index, and the source array.
    
- **`thisArg`:** Optional second argument of `map()`. Sets the `this` context inside a standard function callback. Does not work with arrow functions.