# PostCSS

[What is it?](http://davidwells.io/what-is-postcss/)

## Using Variables

```css
/* define variable */
$variableName: red;

/* usage as css value */
.my-css-class {
  color: $variableName;
}
/* output:
.my-css-class {
  color: red;
}
*/

/* usage as selector */
$variableName { }
// output red { }

/* usage as part of a selector */
$(variableName)_button { }
// output red_button { }
```