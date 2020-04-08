# React Performance Notes

## Code splitting

Code splitting acts on the principle that loading less code will speed up your app.

```jsx
import('/some-module.js').then(
  (module) => {
    // do stuff with the module's exports
  },
  (error) => {
    // there was some error loading the module...
  }
)
```

```jsx
// Code splitting solution
import React from 'react'

const loadTilt = () => import('../tilt')
// Calling React.lazy, it needs to be use with React.Suspense
const Tilt = React.lazy(loadTilt)

function App() {
  const [showTilt, setShowTilt] = React.useState(false)

  React.useEffect(() => {
    loadTilt()
  }, [])

  return (
    <div>
      <label>
        <input
          type="checkbox"
          checked={showTilt}
          onChange={(e) => setShowTilt(e.target.checked)}
        />
        {' show tilt'}
      </label>
      <React.Suspense fallback={<div>loading titl...</div>}>
        {showTilt ? <Tilt>This is tilted!</Tilt> : null}
      </React.Suspense>
    </div>
  )
}
// 🦉 Note that in this app, we actually already have a React.Suspense
// component higher up in the tree where this component is rendered
// (see app.js), so you *could* just rely on that one. Why would you not want
// to do that do you think?

////////////////////////////////////////////////////////////////////
//                                                                //
//                 Don't make changes below here.                 //
// But do look at it to see how your code is intended to be used. //
//                                                                //
////////////////////////////////////////////////////////////////////

function Usage() {
  return <App />
}

export default Usage
```

React has built-in support for loading modules as React components. The module must have a React component as the default export, and you have to use the <React.Suspense /> component to render a fallback value while the user waits for the module to be loaded.

If you're using webpack to bundle your application, then you can use webpack
[magic comments](https://webpack.js.org/api/module-methods/#magic-comments) to
have webpack instruct the browser to prefetch dynamic imports:

## useMemo for expensive calculations

React put all the logic and state management within a function component.

This power comes with an unfortunate limitation that calculations performed within render will be performed every single render, regardless of whether the inputs for the calculations change.

```jsx
// useMemo for expensive calculations
// http://localhost:3000/isolated/exercise/02.js

import React from 'react'
import Downshift from 'downshift'
import { useForceRerender } from '../utils'
import { getItems } from '../filter-cities'

function Menu({
  getMenuProps,
  inputValue,
  getItemProps,
  highlightedIndex,
  selectedItem,
  setItemCount,
}) {
  // Returns a memoized value.
  const items = React.useMemo(() => getItems(inputValue), [inputValue])

  const itemsToRender = items.slice(0, 100)
  setItemCount(itemsToRender.length)
  return (
    <ul
      {...getMenuProps({
        style: {
          width: 300,
          height: 300,
          overflowY: 'scroll',
          backgroundColor: '#eee',
          padding: 0,
          listStyle: 'none',
        },
      })}
    >
      {itemsToRender.slice(0, 100).map((item, index) => (
        <ListItem
          key={item.id}
          getItemProps={getItemProps}
          items={items}
          highlightedIndex={highlightedIndex}
          selectedItem={selectedItem}
          index={index}
        />
      ))}
    </ul>
  )
}
```

Remember that the function passed to `useMemo` runs during rendering. Don’t do anything there that you wouldn’t normally do while rendering. For example, side effects belong in `useEffect,` not `useMemo`.

If no array is provided, a new value will be computed on every render.

- http://kcd.im/usememo

## React.memo for reducing unnecessary re-renders

Here's the lifecycle of a React app:

```
→  render → reconcilitation → commit
         ↖                   ↙
              state change
```

Let's define a few terms:

- The "render" phase: create React elements React.createElement
- The "reconciliation" phase: compare previous elements with the new ones
- The "commit" phase: update the DOM (if needed).

```jsx
// React.memo for reducing unnecessary re-renders
// http://localhost:3000/isolated/exercise/03.js

import React from 'react'
import Downshift from 'downshift'
import { getItems } from '../workerized-filter-cities'
import { useAsync, useForceRerender } from '../utils'

function Menu({
  getMenuProps,
  inputValue,
  getItemProps,
  highlightedIndex,
  selectedItem,
  setItemCount,
}) {
  const { data: items = [] } = useAsync(
    React.useCallback(() => getItems(inputValue), [inputValue])
  )
  const itemsToRender = items.slice(0, 100)
  setItemCount(itemsToRender.length)
  return (
    <ul
      {...getMenuProps({
        style: {
          width: 300,
          height: 300,
          overflowY: 'scroll',
          backgroundColor: '#eee',
          padding: 0,
          listStyle: 'none',
        },
      })}
    >
      {itemsToRender.map((item, index) => (
        <ListItem
          key={item.id}
          getItemProps={getItemProps}
          items={items}
          highlightedIndex={highlightedIndex}
          selectedItem={selectedItem}
          index={index}
        />
      ))}
    </ul>
  )
}
// 🐨 Memoize the Menu here using React.memo

function ListItem({
  getItemProps,
  items,
  highlightedIndex,
  selectedItem,
  index,
}) {
  const item = items[index]
  return (
    <li
      {...getItemProps({
        index,
        item,
        style: {
          backgroundColor: highlightedIndex === index ? 'lightgray' : 'inherit',
          fontWeight:
            selectedItem && selectedItem.id === item.id ? 'bold' : 'normal',
        },
      })}
    >
      {item.name}
    </li>
  )
}
// 🐨 Memoize the ListItem here using React.memo

function FilterComponent() {
  const forceRerender = useForceRerender()

  return (
    <>
      <button onClick={forceRerender}>force rerender</button>
      <Downshift
        onChange={(selection) =>
          alert(
            selection ? `You selected ${selection.name}` : 'Selection Cleared'
          )
        }
        itemToString={(item) => (item ? item.name : '')}
      >
        {({
          getInputProps,
          getItemProps,
          getLabelProps,
          getMenuProps,
          isOpen,
          inputValue,
          highlightedIndex,
          selectedItem,
          setItemCount,
        }) => (
          <div>
            <div>
              <label {...getLabelProps()}>Find a city</label>
              <div>
                <input {...getInputProps()} />
              </div>
            </div>
            <Menu
              getMenuProps={getMenuProps}
              inputValue={inputValue}
              getItemProps={getItemProps}
              highlightedIndex={highlightedIndex}
              selectedItem={selectedItem}
              setItemCount={setItemCount}
            />
          </div>
        )}
      </Downshift>
    </>
  )
}

////////////////////////////////////////////////////////////////////
//                                                                //
//                 Don't make changes below here.                 //
// But do look at it to see how your code is intended to be used. //
//                                                                //
////////////////////////////////////////////////////////////////////

function Usage() {
  return <FilterComponent />
}

export default Usage
```

A React Component can re-render for any of the following reasons:

1. Its props change
2. Its internal state changes
3. It is consuming context values which have changed
4. Its parent re-renders
