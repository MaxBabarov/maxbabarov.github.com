# Anypoint Components 2.0

The purpose of this document is to setup a guideline for creating performant, maintainable, and flexible React components. New components should follow the practices outlined in this document so that the behavior is expected, and consistent across the Anypoint Component library.

Quick reference:

- [JS Guidelines](#js-guidelines)
- [CSS Guidelines](#css-guidelines)
- [Component Guidelines](#component-guidelines)
- [Type Annotation with Flow](#type-annotation-with-flow)


## JS Guidelines

JS linting is powered by ESlint. All JS code should adhere to style guides defined in [eslint-config-mulesoft](https://github.com/mulesoft/javascript).

## CSS Guidelines

We are using PostCSS for all CSS processing. It's very similar to Less and Sass but it allows us to use the future CSS4 spec today.

CSS linting is enabled via [StyleLint](https://github.com/stylelint/stylelint).

Rules:

- Always use local scoped classes; use global classes only when absolutely necessary
- Name your css classes in camelCase so it's easier to reference in JS, and more consistent
- Avoid CSS nesting where possible
- Limit of 3 nested layers of CSS
- Use global variables where possible
- Use global mixins where possible

## Component Guidelines

We are using ES6 class syntax instead of `React.createClass` to create new component classes. Even better, consider using pure functional components if no internal state or lifecycle hooks are needed.

### All components should...

- Support the following standard component props

prop | type | default | description
-----|------|---------|------------
testId | string | `undefined` | adds data-test-id attribute on component, mostly for UI automation tests
className | string | `undefined` | custom class(-es) to apply to the component, will be added to outer most DOM element of the component. To get finer grain customization of components, use `theme`
style | Object | `undefined` | *(Not supported - see `theme`)* inline styles to apply to component
theme | Object | ComponentDefaultTheme | theme object to apply to the component ([Component Theme](#component-theme))


- Have a data attribute `data-anypoint-component` with the value being the name of the component. For example, `<TextField />` should render:

  ```html
  <div data-anypoint-component="TextField" class="...">
    <!-- Implementation details of component -->
  </div>
  ```

- Take a `theme` prop to let users customize the styles of the component. (Learn more about styling components in [Component Theme](#component-theme) section)

- Avoid states as much as possible.

  Use state for something that is really only internal to the component, and will not/rarely need to be controlled from the outside. **Never put `value` in state.** There's a pattern that allows co-existence of state and prop: if prop is undefined, the component controls and uses internal state; if a value is provided to the prop, it will be used instead of the internal state. There's a helper function in utils folder: `propsOrState`.

  ```js
  // example from TableV2
  import propsOrState from '../../utils/propsOrState'
  ...
  // bind function in constructor
  this.propsOrState = propsOrState.bind(this)
  ..
  // in render()
  const sortColumn = this.propsOrState('sortColumn')
  ```

- Keep props minimal, and stick to naming conventions:

	- mirror DOM api as closely as possible
	- self-explanatory
	- use `isLoading` instead of  `loading`
	- use `onSortChange` instead of `sortChangeCb`

### Form components should...

- Support the following standard Form Component props:

prop | type | default | description
-----|------|---------|------------
name | string | `undefined` | key to identify the form field
value | any | `undefined` | value of the form field
disabled | bool | `false` | if the form field is disabled, it should not respond to any user interaction
isValid | bool | `false` | if `true`, the value is valid
isDirty | bool | `false` | if `true`, the input has been touched by the user
isPending | bool | `false` | if `true`, a async validation is still in progress
isFocused | bool | `false` | if `true`, the input is focused

- The following props are used by `<Form/>`, but should be outside of a Form Component's concern.

  For example, label and validations are the form's concern instead of the individual form component's concern. `<TextField />` should merely be a input box without labels or validation tooltip. This makes it much easier to compose components. (Timepicker = Select + TextField)

prop | type | default | description
-----|------|---------|------------
align | string | `"vertical"` | alignment of form fields and labels
label | string | `undefined` | string to show as label for the form field
validations | Object[] | `[]` | validators to validate the value of the form field
validateOn | string | `"blur"` | when to perform the validation
required | bool | `false` | whether the form field is required or not
showValidation | bool | `false` | if `true`, validation info will be shown for this form field


- Should implement the following behavior:

	- When value changed due to user interaction, call `onChange(newValue)`
	- When component is focused, call `onFocus(event)`
	- When component is blurred, call `onBlur(event)`


- Should be wrapped in a `<div>` tag, and should not take a `width` prop. By default, Form component should take up the entire width of the parent container. To control the dimension of a form component, either style the parent container, or use CSS and className to style the component.

### Component Theme

Add styles to your component through CSS, follow the general guidelines as metioned in [CSS Guidelines](#css-guidelines) section, but instead of applying classes directly, use [react-css-themr](https://github.com/javivelasco/react-css-themr).

```js
// regular CSS modules
import styles from 'Pill.css'
// ...
render() {
  return (
    <div className={styles.pill}>
      <button className={styles.button}/>
    </div>
  )
}

// =================================

// react-css-themr
import { themr } from 'react-css-themr'
import themeCSS from 'Pill.css'
// ...
render() {
  const { theme } = this.props
  return (
    <div className={theme.pill}>
      <button className={theme.button}/>
    </div>
  )
}
// ...
export default themr('Pill', themeCSS)(Pill);
```

Both methods look almost identical in terms of code, with the themr method requiring a few extra lines. However, react-css-themr gives the component ability to support themes, which not only allows more fine-grained control of styles internal to the component, it also allows the user to apply custom styling to components across the application (through context), or simple overwrite styles of a single instance (through prop). For more information, read about how [react-css-themr works](https://github.com/javivelasco/react-css-themr#how-does-it-work).

### Methods in components

Function names should be prefixed with `handleX` if being attached to an eventListener `onX`.

Example: `handleClick` is what you name the function that you pass to the `onClick` prop.

```js
// Bad
doMyCustomStuff() {}
render() {
  return <button onClick={this.doMyCustomStuff}>Click me</button>;
}

// Good
handleClick() {}
render() {
  return <button onClick={this.handleClick}>Click me</button>;
}
```

### Performance standards
- Use `shouldComponentUpdate` where possible. [Why?](https://facebook.github.io/react/docs/advanced-performance.html#shouldcomponentupdate-in-action)
- Avoid instantiating default values in `render()` (especially object + arrays)

```js
// Bad. Array instantiated every render and CustomComponent optimizations not possible
class ComponentExample extends Component {

  render(){
     const list = this.props.list || [];
     return(<CustomComponent list={list}></div>)
  }

}

// Good. defaultList will always === defaultList and shouldComponentUpdate will work on equality checks
const defaultList = [];
class ComponentExample extends Component {

  render(){
    const list = this.props.list || defaultList;
    return(<CustomComponent list={list}></div>)
  }

}
```

- All event handlers must be bound in component class constructor.  [See why this is important](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-no-bind.md)

```js
// Bad. Bind handlers in render
import React, {PropTypes, Component} from 'react';

class ComponentExample extends Component {

  constructor(props) {
    super(props)
  }
  handleClick(){
    // do stuff
  }
  render(){
    return(<div onClick={this.handleClick.bind(this)}></div>)
  }

}

// Good. Bind handlers once in constructor
import React, {PropTypes, Component} from 'react'

class ComponentExample extends Component {

  constructor(props) {
    super(props)
    this.handleClick = this.handleClick.bind(this)
  }
  handleClick(){
    // do stuff
  }
  render(){
    return(<div onClick={this.handleClick}></div>)
  }

}
```

## Type Annotation with Flow

We use [Flow](https://flowtype.org/) for javascript type annotation, which does type inference and type checking during development time. It replaces React Proptype checks, and also comes with IDE integrations/plugins for Vim, Emacs, and Atom([Nuclide](http://nuclide.io/docs/languages/flow/)).

General guidelines for doing type annotation with flow, all of the following rules are checked by eslint with configuration defined in `config/eslint.js`:

```js
// annotate your function parameters
function sum(a: number, b: number) {
  return a + b;
}


// put a space after the colon, before type annotation
// Good
const x: string = 'Hello'
// Bad
const x:   string = 'Hello'
// Bad
const x:string = 'Hello'


// no spaces before the colon
// Good
const x: string = 'Hello'
// Bad
const x : string = 'Hello'


// no spaces before generic type bracket
// Good
const x: Array<number> = [1, 2]
// Bad
const x: Array <number> = [1, 2]

// Naming conventions
// Good
type Props = {...};
type DefaultProps = {...};
type State = {...};
// Suffix user defined types with 'T', otherwise it's easy to confuse it with variables and classes
type SizeT = 's' | 'm' | 'l';
type TextFieldTypeT = 'text' | 'password' | 'email';

// Bad
type Size = 's' | 'm' | 'l';
```
