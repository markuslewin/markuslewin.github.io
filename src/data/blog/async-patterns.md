---
title: 'Async UI patterns with React'
description: |
  Exploring the new primitives for async work in modern React.
publishedAt: '2026-03-05T00:02+00:00'
---

I recently watched the talk "[Beyond Signals](https://www.youtube.com/watch?v=DZPSAOBnBAM)" by Ryan Carniato. The talk is about reactivity in frontend frameworks. Ryan mentions [four patterns](https://www.youtube.com/watch?v=DZPSAOBnBAM&t=676s) of "async consistency" in user interfaces. He also discusses these patterns in depth on his [streams](https://www.youtube.com/@ryansolid/streams) on the subject.

Building async UI is not new to me. I had never thought of the solutions boiling down to only four patterns, though. React 19 introduced new features for managing async work. I have used some of those features in Next. However, I wanted to see how the APIs would come together using only React.

To do this, I decided to implement the four patterns using modern React features.

React 19 has been out for a while. However, [there are still no Learn docs](https://github.com/reactwg/async-react/discussions/2) explaining related concepts. Instead, the talk "[Async React](https://www.youtube.com/watch?v=B_2E96URooA)" by Ricky Hanlon serves as an introduction to the features. I used the API reference for each method to guide me through the exercise.

## Sync Execution

Modern frameworks guarantee consistency in our apps. They avoid "tearing" the UI.

```jsx
const Component = () => {
	const [count, setCount] = useState(0)

	const doubleCount = count * 2

	return (
		<>
			{count} * 2 = {doubleCount}
			<button
				onClick={() => {
					setCount(count + 1)
				}}
			>
				Increment count
			</button>
		</>
	)
}
```

React synchronizes the values of `count` and `doubleCount`. When the user presses the button, React does not update the UI to `1 * 2 = 0`. Instead, it recalculates the value of `doubleCount` before updating the UI. This keeps the UI consistent.

This is easy enough in sync execution. However, what happens if the act of calculating `doubleCount` requires async work? How would we want our UI to behave when the user clicks the button?

## Async Patterns

Ryan mentions four patterns of async consistency:

- Show Placeholder
- Hold in the Past
- Show the Future
- Tear

The following table describes the behavior of the values for each pattern:

<div class="overflow-x-auto">

| Name             | Value A | Value B |
| ---------------- | ------- | ------- |
| Hold in the Past | Stale   | Stale   |
| Show Placeholder | Updated | None    |
| Tear             | Updated | Stale   |
| Show the Future  | Updated | Updated |

</div>

If we see "Value A" as `count` and "Value B" as `doubleCount`, we can add a new column. This column will show how our UI looks after the user increments the value of `count`:

<div class="overflow-x-auto">

| Name             | `count` | `doubleCount` | UI                   |
| ---------------- | ------- | ------------- | -------------------- |
| Hold in the Past | Stale   | Stale         | `0 * 2 = 0`          |
| Show Placeholder | Updated | None          | `1 * 2 = Loading...` |
| Tear             | Updated | Stale         | `1 * 2 = 0`          |
| Show the Future  | Updated | Updated       | `1 * 2 = 2`          |

</div>

This simple example highlights the [main characteristics of each pattern](https://www.youtube.com/live/eZAcJc_eR1M?si=UMlzzUp7PEcibYwT&t=1279).

"Hold in the Past" is the only pattern with no immediate feedback. The UI stays consistent, but it appears to freeze every time the user presses the button. When using this approach, it is crucial that we add a pending state that shows data is loading.

"Show Placeholder" is the most truthful of the four. `count` updates, but the app does not know the next value of `doubleCount`. Consequently, the UI replaces the value of `doubleCount` with a placeholder. This placeholder usually comes in the form of a "spinner". Replacing already visible content can result in a jarring user experience.

"Tear" provides immediate feedback as `count` updates. The drawback of this is that the UI is inconsistent. We want to avoid this strategy for mathematical expressions. However, there are situations where it is okay to combine updated and stale values.

"Show the Future" - or "optimistic UI" - makes the UI behave as if the process were sync. This works well when we can make a correct guess for the next value. We also have to consider the user experience for incorrect guesses. A server might fail to persist an update, for example.

All these patterns are valid. We choose the most suitable one for our use case.

These are _UI_ patterns, not _implementation_ patterns. There are several ways of implementing each pattern, even in the same framework!

## Consuming Promises in React

JavaScript uses Promises to model async work. To manage async work in React, we need the ability to wait for a Promise to resolve. This is what the [`use`](https://react.dev/reference/react/use) method is for:

```jsx
const Component = ({ promise }) => {
	const value = use(promise)
	return <>Resolved with: {value}</>
}
```

While the Promise read by `use` is pending, `use` "suspends" the component, and in turn, the whole UI. We want to avoid freezing the whole UI. We can use a [`Suspense`](https://react.dev/reference/react/Suspense) boundary to localize the pending state. This keeps the rest of our UI interactive. We can also add an error boundary to display an error message in case the Promise rejects:

```jsx
const App = () => {
	return (
		<ErrorBoundary fallback={'Rejected!'}>
			<Suspense fallback={'Pending...'}>
				<Component promise={promise} />
			</Suspense>
		</ErrorBoundary>
	)
}

const Component = ({ promise }) => {
	const value = use(promise)
	return <>Resolved with: {value}</>
}
```

This is the equivalent imperative logic:

```js
try {
	const value = await promise
	console.log('Resolved with:', value)
} catch {
	console.log('Rejected!')
}
```

When `use` reads a pending Promise, it suspends the component. This causes the Suspense component to display its "Pending..." fallback. When the Promise resolves, our `Component` renders its message containing the resolved value.

## Implementing the Patterns

### Show Placeholder

We can implement the "Show Placeholder" pattern with Suspense. The component provided to the `fallback` prop acts as the placeholder.

```jsx
const App = () => {
	return (
		<Suspense fallback={<CardsSkeleton />}>
			<Cards />
		</Suspense>
	)
}

const Cards = () => {
	const cards = use(getCards())

	return cards.map((card) => {
		return <Card data={card} />
	})
}
```

A common use case for this pattern is to show "skeletons" for initial page loads. With [Next's loading.js feature](https://nextjs.org/docs/app/api-reference/file-conventions/loading), the user receives a cached layout. Then, Next fills the layout with data as it becomes available.

### Hold in the Past

Pulling the UI from under the user can result in a jarring user experience. For a smoother transition between states, we can "Hold in the Past".

```jsx
const starters = [
	{ no: 1, name: 'Bulbasaur' },
	{ no: 4, name: 'Charmander' },
	{ no: 7, name: 'Squirtle' },
]

const App = () => {
	const [selectedId, setSelectedId] = useState(0)
	const [isPending, startTransition] = useTransition()

	return (
		<>
			<div>
				{starters.map((starter) => {
					return (
						<button
							key={starter.no}
							onClick={() => {
								startTransition(() => {
									selectedNo(starter.no)
								})
							}}
						>
							{starter.name}
						</button>
					)
				})}
			</div>
			<Suspense fallback={<PokemonSkeleton />}>
				<div data-is-pending={isPending}>
					<Pokemon no={no} />
				</div>
			</Suspense>
		</>
	)
}

const Pokemon = ({ no }) => {
	const pokemon = use(getPokemon(no))

	return (
		<div>
			<h2>{pokemon.name}</h2>
			<p>{pokemon.description}</p>
		</div>
	)
}
```

Here, I am implementing the pattern using React's [Transitions](https://react.dev/reference/react/useTransition). When state updates inside a Transition, React renders the next UI _in the background_. The Transition stays in a pending state while components are suspending. In other words, the Transition is pending while the new Promises are pending.

While this is happening, the app shows the user the stale UI. We can fade the content or show a complementary spinner to tell the user that the next state is loading.

### Tear

Tearing the UI is often something we want to _avoid_ doing. However, there are some situations where it can work well for async operations.

One of those situations is when the user is typing into a search field. The app tries to provide suggestions, but it takes time to fetch that data. Here, stale suggestions are better than no suggestions. The UI is tearing; the text field contains fresh data, but the suggestions are stale.

Here, I am using Transitions in combination with an [optimistic](https://react.dev/reference/react/useOptimistic) value. While a Transition is _not_ pending, the value of the optimistic term is the same as the ordinary term. While a Transition _is_ pending, the value of the optimistic term is the value set using `setOptimisticTerm`.

```jsx
const App = () => {
	const [term, setTerm] = useState('')
	const [optimisticTerm, setOptimisticTerm] = useOptimistic(term)
	const [isPending, startTransition] = useTransition()

	return (
		<>
			<input
				name="term"
				placeholder="Search a pokémon"
				value={optimisticTerm}
				onChange={(event) => {
					startTransition(() => {
						const nextTerm = event.target.value
						setOptimisticTerm(nextTerm)
						setTerm(nextTerm)
					})
				}}
			/>
			<Suspense fallback={<p>Loading...</p>}>
				<SearchResult term={term} />
			</Suspense>
		</>
	)
}

const SearchResult = ({ term }) => {
	const pokemons = use(getPokemons(term))

	return (
		<ol>
			{pokemons.map((pokemon) => {
				return <li key={pokemon.no}>{pokemon.name}</li>
			})}
		</ol>
	)
}
```

The [React docs for `useDeferredValue`](https://react.dev/reference/react/Suspense#showing-stale-content-while-fresh-content-is-loading) contain a similar example. In the docs, `useDeferredValue` introduces staleness. In my implementation, the suggestions are stale because of Transitions. In the docs, the text field updates through an ordinary state setter. In my implementation, the text field updates through an optimistic value.

The async UI patterns dictate user experiences—not implementations.

### Show the Future

For some user experiences, we can "Show the Future". We guess what the UI will look like after the async process completes. This makes an async process seem sync. It is common to use this pattern when the user performs an async mutation.

Earlier, I used the _properties_ of optimistic values to update a text field. The update was not optimistic; I _knew_ what the next state would look like.

Here, I'm being "truly optimistic". I don't actually know what value the server is going to respond with after the user presses the "Like" button.

```jsx
const App = () => {
  const [isLiked, postLikeAction] = useActionState((_, nextIsLiked) => {
    return await postIsLiked(nextIsLiked)
  }, false);
  const [optimisticIsLiked, setOptimisticIsLiked] = useOptimistic(isLiked);

  return (
    <button
      onClick={() => {
        startTransition(() => {
          const nextIsLiked = !optimisticIsLiked
          setOptimisticIsLiked(nextIsLiked)
          postLikeAction(nextIsLiked)
        });
      }}
    >
      {isLiked ? "Unlike" : "Like"}
    </button>
  );
};
```

We could have implemented this feature with ordinary `useState` and raw `fetch`es. The problem with that approach is that it is sensitive to race conditions. The response order might be different from the request order.

`useActionState` solves this problem by queuing the requests. The app does not make a second request before it has received a response for the first request, and so on.

## Conclusion

That is the four async patterns implemented using new React features!

Transitions **hold in the past** to prevent jarring **placeholders**. I can orchestrate Transitions to **tear** or **show the future** using optimistic values.
