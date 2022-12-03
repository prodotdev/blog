---
title: Finally Understanding useEffect, useCallback, and useMemo
date: "2022-12-03T00:00:00.000Z"
---

When I first started learning React, I found these 3 hooks the hardest to wrap my head around. The docs, though extensive, focuses more on the caching benefits of these three hooks, and doesn't really provide a simple example of when you _must_ use these hooks. If you've been struggling to really understand when you *need* these hooks, hang around, and I'll try break it down as I simply can.

###### Takeaway #1 - A React component is a function.

Everytime something changes, the function is run. Specifically, each time a prop changes, or a `setState` is called, the entire function is called **again**.

```js
function MyComponent() {
   const [count, setCount] = useState(0)

   console.log('called')
   
   return (<button onClick={() => setCount(count + 1)}>click</button>) 
}
```

Each time you click the button, you’ll see `called` printed the console, but what if we didn't want to print `called` every single click? If we only want to do something sometimes, and not each time the component is run, we can use `useEffect`.

```js
function MyComponent() {
   const [count, setCount] = useState(0)

   useEffect(() => {
      console.log('called')
   }, [])
   
   return (<button onClick={() => setCount(count + 1)}>click</button>) 
}
```

`called` will now only be printed once when the component first renders, and most importantly, each click will **not** print `called`.

###### Takeaway #2 - Use `useEffect` for things that you only want to run sometimes.

We can go one step further. If we want the `useEffect` to be run whenever some variable changes, we can add it to the dependencies array: `[]`. 

```js
function MyComponent() {
   const [count, setCount] = useState(0)

   useEffect(() => {
      console.log('called')
   }, [count])
   
   return (<button onClick={() => setCount(count + 1)}>click</button>) 
}
```

Here, we've added `count` to the dependencies array, so now whenever the value changes (on every button click), we see `called` being printed to the console.

###### Takeaway #3 - When something changes in the array, the `useEffect` is called again.

Ok, now what if instead of whenever a variable changes, we want to re-run `useEffect` whenever a _function_ changes, let's try it out:

```jsx
function MyComponent() {
   const [count, setCount] = useState(0)

   const logCalled = () => { console.log('called') }

   useEffect(() => {
      logCalled()
   }, [logCalled])
   
   return (<button onClick={() => setCount(count + 1)}>click</button>) 
}
```

Since we're only defining `logCalled` once, and we never update it, it should only be called once, right? Nope. 

What will end up happening is, every time we click the button `called` will be printed, as if `logCalled` was a different function - but we never updated the function, what's going on? This is because in JS functions are objects, and objects do not equal `===` objects. That is:

```js
const a = { name: "Mike" }
const b = { name: "Mike" }
```

These two objects might look identical, but if you do `a === b` it will always return `false`. This is basically what’s happening above, React checks each time if `logCalled === logCalled`  and it returns false because it's redefined, and so it re-runs the `useEffect`.

###### Takeaway #4 - Each time the component function is called, every object, or function is a new, separate object.

To prevent React from defining a new function we need to use `useCallback`:

```jsx
function MyComponent() {
   const [count, setCount] = useState(0)

   const logCalled = useCallback(() => { console.log('called') }, [])

   useEffect(() => {
      logCalled()
   }, [logCalled])
   
   return (<button onClick={() => setCount(count + 1)}>click</button>) 
} 
```

When we call `useCallback` we're telling React to give us the same function, in this case `() => { console.log('called') }`. Now `useEffect` sees that the `logCalled` is the same function, and it won't get called again.

You may have noticed that `useCallback` has a `[]` dependencies array too. Similar to `useEffect`, we can tell React we _do_ want a new function returned whenever some variable changes.

Got it? Ok, cool, now let's look at `useMemo`. We just learnt we can use `useCallback` for functions, similarly for regular objects, we'd use `useMemo`:


```jsx
function MyComponent() {
   const [count, setCount] = useState(0)

   const author = useMemo(() => {
    return {
      name: "Mike"
    }
   }, [])

   useEffect(() => {
    console.log(author.name)
   }, [author])
   
   return (<button onClick={() => setCount(count + 1)}>click</button>) 
} 
```

If we didn't use `useMemo` to return an object, we'd see the author's name printed, on every button click.

###### Takeaway #5 - Use `useMemo` for objects, and `useCallback` for functions.

Ok, so now that we know _why_ we need them, let's recap on _when_ to use them. **It all starts with `useEffect`.** If you only need to do something _sometimes_, you need `useEffect`. Things like:

- Making an API call to fetch a user.
- Log some data.
- Animate UI

You'll see these actions being referred to as *side-effects*. So whenever you need to perform a side-effect, you need a `useEffect`. See where the name comes from? When you use a `useEffect`, **always** define a dependency array: `[]`, and add all your variables.

For every dependency you add to the array:

**1. Is it a string, number, boolean?** Yes, you're done.

**2. Is it an object?** Wrap in `useMemo`.

**3. Is it a function?** Wrap in `useCallback`.

**4. Does the `useMemo` / `useCallback` have any dependencies?** No, you're done.

**5. Go through each dependency, and start back at 1.**

###### Takeway #6 - _Every_ dependency that is an object, or function, should _always_ be wrapped in either a `useMemo`, or` useCallback`.

Now you may be thinking, why don't I just wrap all object, and functions when I write them? Yea, don't do that. The extra time you spend typing, and any performance gains there might be is NOT worth the bigger, messier, components.

My approach is to just not worry about any of them, write your components, until you add a `useEffect`. That should be your queue to start going through the dependencies.