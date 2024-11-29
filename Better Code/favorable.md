# favorable

```js
// --run--
const favorable = (i) => {
  return '☆☆☆☆☆★★★★★'.slice(i, 10 - (5 - i))
};

console.log(favorable(1));
console.log(favorable(2));
console.log(favorable(3));
console.log(favorable(4));
console.log(favorable(5));
```
