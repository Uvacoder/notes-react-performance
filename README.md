# React Performance Notes

# 1. Code splitting

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

# 2. useMemo for expensive calculations

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

# 3. React.memo for reducing unnecessary re-renders

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

# 4. Window large lists with react-window

> NOTE: We’re going to be using a library called react-window.

```jsx
// Window large lists with react-window
// http://localhost:3000/isolated/final/04.js

import React from 'react'
import Downshift from 'downshift'
import { FixedSizeList as List } from 'react-window'
import { getItems } from '../workerized-filter-cities'
import { useAsync, useForceRerender } from '../utils'

function Menu({
  getMenuProps,
  inputValue,
  getItemProps,
  highlightedIndex,
  selectedItem,
  setItemCount,
  listRef,
}) {
  const { data: items = [] } = useAsync(
    React.useCallback(() => getItems(inputValue), [inputValue])
  )
  setItemCount(items.length)
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
      <List
        ref={listRef}
        width={300}
        height={300}
        itemCount={items.length}
        itemSize={20}
        itemData={{
          getItemProps,
          items,
          highlightedIndex,
          selectedItem,
        }}
      >
        {ListItem}
      </List>
    </ul>
  )
}
Menu = React.memo(Menu)

function ListItem({
  data: { getItemProps, items, highlightedIndex, selectedItem },
  index,
  style,
}) {
  const item = items[index]
  return (
    <li
      {...getItemProps({
        index,
        item,
        style: {
          ...style,
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

function FilterComponent() {
  const forceRerender = useForceRerender()
  const listRef = React.useRef()

  function handleStateChange(changes, downshiftState) {
    if (changes.hasOwnProperty('highlightedIndex') && listRef.current) {
      listRef.current.scrollToItem(changes.highlightedIndex)
    }
  }

  return (
    <>
      <button onClick={forceRerender}>force rerender</button>
      <Downshift
        onStateChange={handleStateChange}
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
              listRef={listRef}
            />
          </div>
        )}
      </Downshift>
    </>
  )
}

function Usage() {
  return <FilterComponent />
}

export default Usage

/*
eslint
  no-func-assign: 0,
*/
```

# 5. Fix “perf death by a thousand cuts”

When you’re building a sizable real-world application, you’re typically going to need some sort of state management solution. Whatever state management solution you’re using, often you can run into a problem that I call "perf death by a thousand cuts" which basically means that so many components are updated when state changes that it becomes a performance bottleneck.

```jsx
// Solution:
// Fix "perf death by a thousand cuts"
// http://localhost:3000/isolated/final/05.js

import React from 'react'
import useInterval from 'use-interval'
import { useForceRerender, useDebouncedState } from '../utils'

const AppStateContext = React.createContext()

// increase this number to make the speed difference more stark.
const dimensions = 100
const initialGrid = Array.from({ length: dimensions }, () =>
  Array.from({ length: dimensions }, () => Math.random() * 100)
)

const initialRowsColumns = Math.floor(dimensions / 2)

function appReducer(state, action) {
  switch (action.type) {
    case 'UPDATE_GRID': {
      return {
        ...state,
        grid: state.grid.map((row) => {
          return row.map((cell) =>
            Math.random() > 0.7 ? Math.random() * 100 : cell
          )
        }),
      }
    }
    default: {
      throw new Error(`Unhandled action type: ${action.type}`)
    }
  }
}

function AppStateProvider({ children }) {
  const [state, dispatch] = React.useReducer(appReducer, {
    grid: initialGrid,
  })
  const value = [state, dispatch]
  return (
    <AppStateContext.Provider value={value}>
      {children}
    </AppStateContext.Provider>
  )
}

function useAppState() {
  const context = React.useContext(AppStateContext)
  if (!context) {
    throw new Error('useAppState must be used within the AppStateProvider')
  }
  return context
}

function UpdateGridOnInterval() {
  const [, dispatch] = useAppState()
  useInterval(() => dispatch({ type: 'UPDATE_GRID' }), 500)
  return null
}
UpdateGridOnInterval = React.memo(UpdateGridOnInterval)

function ChangingGrid() {
  const [keepUpdated, setKeepUpdated] = React.useState(false)
  const [state, dispatch] = useAppState()
  const [rows, setRows] = useDebouncedState(initialRowsColumns)
  const [columns, setColumns] = useDebouncedState(initialRowsColumns)
  const cellWidth = 40
  return (
    <div>
      <form onSubmit={(e) => e.preventDefault()}>
        <div>
          <button
            type="button"
            onClick={() => dispatch({ type: 'UPDATE_GRID' })}
          >
            Update Grid Data
          </button>
        </div>
        <div>
          <label htmlFor="keepUpdated">Keep Grid Data updated</label>
          <input
            id="keepUpdated"
            checked={keepUpdated}
            type="checkbox"
            onChange={(e) => setKeepUpdated(e.target.checked)}
          />
          {keepUpdated ? <UpdateGridOnInterval /> : null}
        </div>
        <div>
          <label htmlFor="rows">Rows to display: </label>
          <input
            id="rows"
            defaultValue={rows}
            type="number"
            min={1}
            max={dimensions}
            onChange={(e) => setRows(e.target.value)}
          />
          {` (max: ${dimensions})`}
        </div>
        <div>
          <label htmlFor="columns">Columns to display: </label>
          <input
            id="columns"
            defaultValue={columns}
            type="number"
            min={1}
            max={dimensions}
            onChange={(e) => setColumns(e.target.value)}
          />
          {` (max: ${dimensions})`}
        </div>
      </form>
      <div
        style={{
          width: '100%',
          maxWidth: 410,
          maxHeight: 820,
          overflow: 'scroll',
        }}
      >
        <div style={{ width: columns * cellWidth }}>
          {state.grid.slice(0, rows).map((row, i) => (
            <div key={i} style={{ display: 'flex' }}>
              {row.slice(0, columns).map((cell, cI) => (
                <Cell key={cI} cellWidth={cellWidth} cell={cell} />
              ))}
            </div>
          ))}
        </div>
      </div>
    </div>
  )
}
ChangingGrid = React.memo(ChangingGrid)

function Cell({ cellWidth, cell }) {
  return (
    <div
      style={{
        outline: `1px solid black`,
        display: 'flex',
        justifyContent: 'center',
        alignItems: 'center',
        width: cellWidth,
        height: cellWidth,
        color: cell > 50 ? 'white' : 'black',
        backgroundColor: `rgba(0, 0, 0, ${cell / 100})`,
      }}
    >
      {Math.floor(cell)}
    </div>
  )
}
Cell = React.memo(Cell)

function DogNameInput() {
  const [dogName, setDogName] = React.useState('')

  function handleChange(event) {
    const newDogName = event.target.value
    setDogName(newDogName)
  }

  return (
    <form onSubmit={(e) => e.preventDefault()}>
      <label htmlFor="dogName">Dog Name</label>
      <input
        value={dogName}
        onChange={handleChange}
        id="dogName"
        placeholder="Toto"
      />
      {dogName ? (
        <div>
          <strong>{dogName}</strong>, I've a feeling we're not in Kansas anymore
        </div>
      ) : null}
    </form>
  )
}

function App() {
  return (
    <div>
      <DogNameInput />
      <AppStateProvider>
        <ChangingGrid />
      </AppStateProvider>
    </div>
  )
}

function Usage() {
  const forceRerender = useForceRerender()
  return (
    <div>
      <button onClick={forceRerender}>force rerender</button>
      <App />
    </div>
  )
}

export default Usage

/*
eslint
  no-func-assign: 0,
*/
```

# 6. Optimize context value

This can be problematic in certain scenarios. You can read more about this here: https://github.com/kentcdodds/kentcdodds.com/blob/319db97260078ea4c263e75166f05e2cea21ccd1/content/blog/how-to-optimize-your-context-value/index.md

The quick and easy solution to this problem is to memoize the value that you provide to the context provider:

```jsx
const CountContext = React.createContext()

function CountProvider(props) {
  const [count, setCount] = React.useState(0)
  const value = React.useMemo([count, setCount], [count])
  return <CountContext.Provider value={value} {...props} />
}
```
