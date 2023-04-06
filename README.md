# code-review-tips

## Table of Contents

1. [Introduction](#introduction)
2. [Why Review Code?](#why-review-code)
3. [Basics](#basics)
4. [Readability](#readability)
5. [Side Effects](#side-effects)
6. [Limits](#limits)
7. [Security](#security)
8. [Performance](#performance)
9. [Testing](#testing)
10. [Miscellaneous](#miscellaneous)

## Introduction

The problem with React, is that it is opinionated, yet provides many ways to solve
the same problem. That allow developers with plenty of room to screw things up with
our own stupid ideas. Here are some anti-patterns and tips and tricks to improve our code.

## Functional Components > Class Components

Use Functional Components instead of Class Based Components. Functional Components
have a much better state management mechanism. In React 16.8 hooks were announced,
and they create reactive state management model right into React. Using `useState` and `useReducer` you declare your state, and using `useEffect`, `useCallback` and `useMemo` you react to changes in that state. And this few hooks created a ecosystem of awesome custom hooks. There are hooks to get access to API, animations, saving data in localStorage and more.

## Functional Components are not Templates

Don’t think of Functional Components as templates. This piece of code cause an infinite loop. `setUsers` will compare previous value and new one,by using `===` operator, and if it were number or a string, it may work, but in this case, with an array or objects, JavaScript will compare not by value, but by reference, and in this case, it will not match, and the state will be updated.
State update will cause another render of `UserList` function, and fetch will run again, and so on and so on.

```javascript
const UserList = () => {
  const [users, setUsers] = useState([]);

  fetch('api/users')
    .then((res) => res.json())
    .then(setUsers);

  return (
    <div>
      {users.map((user) => {
        return <div>{user.name}</div>;
      })}
    </div>
  );
};
```

The fix would be wrapping `fetch` in `useEffect` with `[]` array of dependencies.
This will tell React to run this `fetch` only once, when component gets mounted. Good mindset
will be to breakdown function into parts: Hooks and JSX. And the rule is anything in the Hooks
section should probably be wrapped in a hook, unless you absolutely know that it shouldnt.

```javascript
const UserList = () => {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    fetch('api/users')
      .then((res) => res.json())
      .then(setUsers);
  }, []);

  return (
    <div>
      {users.map((user) => {
        return <div>{user.name}</div>;
      })}
    </div>
  );
};
```

## Dont use big components, break down into smaller pieces, but not too much

BLA BLA BLA

## Use TypeScript

TypeScript helps you to make more robust and reliable applications. There are two times when TypeScript generally touches the React application. The first one is when you defining your types, the kind of data shapes that comeback from the server. The over time is when you are defining your React components

Types

```typescript
interface Person {
  id: string;
  name: string;
  email: string;
  address?: {
    stree: string;
    city: string;
  };
}

interface GetPeopleResponse {
  page: number;
  people: Person[];
  lastUrl: string;
  nextUrl: string;
}
```

Components

```typescript
interface MyListProps {
  list: Person[];
  onClick: (person: Person) => void;
}

const MyList = ({ list, onClick }: MyListProps) => {
  return <div></div>;
};
```

## Don’t worry about re-renders

Whenever you return as JSX, is actually created using `createElement` method, which
eventually returns a Virtual DOM Node and adds it to Virtual DOM Tree, which is in-memory representation of actual DOM Tree should look like. And it is up to React to traverse that Virtual DOM Tree and then synchronize it with the real DOM Tree. It creates, deletes and updates. Nodes, but if there are no changes, then nothing happens.

```typescript
const Title = ({ caption }: { caption: string }) => {
  <h2>{caption}</h2>;
  // Virtual DOM Node
  // React.createElement('h2', null, title);
};
```

If you are getting a lot of re-renders, it is probably just a bug, probably a `useEffect` that has gone into an infinite loop, because of bad dependency array. But if you are legitemaly seeing perfroamnce problems, there are a lot of tools that help you diagnose and fix them. But a strong reccomendtation will be is don’t fight the freamwork, dont try to prematurely optimise your applicaiton by worrying so much about whever or not specific components are re-rendering

## Learn to love dependency arrays

Dependency arrays are the arrays at the end of useEffect, useMemo and useCallback. They tell React when, for example your useEffect should run. So if any items in that [] are changed, then your hook re-runs.

This useEffect runs only ones, even if `id` was `null` and then got the number, it will not re-run, because it already ran with `null` as option

```javascript
useEffect(() => {
  fetch(`api/user/${id}`)
    .then((res) => res.json())
    .then(setUser);
}, []);
```

So what we want to do, is to add anything this useEffect is reading from into the dependency array.

```javascript
useEffect(() => {
  fetch(`api/user/${id}`)
    .then((res) => res.json())
    .then(setUser);
}, [id]);
```

But be careful adding all of things, because you can get into an infinite loop. Take a look at this example. Whenever this useEffect completes, it changes the `user`, which is specified in `[]` and it cause another render, and you get into an infinite loop

```javascript
useEffect(() => {
  fetch(`api/user/${id}`)
    .then((res) => res.json())
    .then(setUser);
}, [id, user, setUser]);
```

Dont disable Linting, because in different cases it can help and suggest what things to add and what to remove from that `[]`.

Another thing to watch for in `[]` is when you have there an array or an object or a function, because React uses the same logic that it uses in that state setter to decide whether the value is the same or different, between the old and new one. For strings, booleans and numbers it does it by value, which is very predictible. However when it comes to arrays, objects and functions it doesn’t look at the contents, it doesn’t do a deep compare, it instead does a referential
compare: is this exactly the same array. So be careful with that.

## Don’t ignore useCallback or useMemo

These hooks are vital to react state management model. If they are used properly, to retain referential identity, they can be a performance enhancement. Memoize === remember.

Two rules to use useMemo:

- Rule 1 - you are doing operation that is going to be expensive
- Rule 2 - you are computing an array or an object, because those are maintained by reference

Rule 1

```javascript
const total = useMemo(() => {
  return costs.reduce((acc, cur) => acc + cur, 0);
}, []);
```

Rule 1 and Rule 2

```javascript
const sortedPeople = useMemo(() => {
  return [...people].sort();
}, [people]);
```

No need to use useMemo (no rules imply)

```javascript
const fullName = () => {
  return `${first} ${last}`;
};
```

useCallback is also good in two cases:

- Rule 1 - if you want to keep your callback functions from being stale
- Rule 2 - if you want to retain the referential identity of those callbacks

As you can see, everytime component re-renders, we create a new reference to a new version of that `sortFunc` function. The implementation is exactly the same, every single time, but dependency arrays dont look at the implementation of the function, they look at the reference, and this going to invalidate that `useMemo` and it going to sort on every render, even though names hasn’t changed, but the `sortFunc` has and that is way it the `useMemo` run.

```javascript
const NameList = ({ names, sortFunc }) => {
  const sortedNames = useMemo(() => {
    return [...names].sort();
  }, [names, sortFunc]);

  return <div>{sortedNames.map()}</div>;
};

<NamesList names={names} sortFunc={() => {}} />;
```

How to get around it? Just use `useCallback` and pass it on. Now we can be sure that

every single time that we call that NamesList components, we are using exactly the same sort

function and therefore that useMemo will not re-run each time.

```javascript
const sortFunc = useCallback((a, b) => .... []);

<NamesList names={names} sortFunc={sortFunc} />
```

## Use Custom Hooks

Make your own custom hooks. They are a collections of hooks, gathered together as a function, that acomplishes a specific task.

```javascript
import { useEffect, useState } from 'react';

const useFetchPosts = () => {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    fetch('https://jsonplaceholder.typicode.com/posts')
      .then((response) => response.json())
      .then((json) => setPosts(json));
  }, []);

  return posts;
};

const FeaturedPosts = () => {
  const posts = useFetchPosts();
  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
};
```

## Create a system of proper resuing of components

```javascript
// Bad: Component is not extendable
const Thingie = ({ description }: { description: string }) => {
  <div className='thingie'>
    <div className='description-wrapper'>
      <Description value={description} />
    </div>
  </div>;
};

const ThingieWithTitle = ({
  title,
  description,
}: {
  title: string,
  description: string,
}) => {
  <div className='thingie'>
    <Title value={title} />
    <div className='description-wrapper'>
      <Description value={description} />
    </div>
  </div>;
};

// Good: Component is extendable and scalable
const Thingie = ({
  description,
  children,
}: {
  description: string,
  children: ReactNode,
}) => {
  <div className='thingie'>
    {children}
    <div className='description-wrapper'>
      <Description value={description} />
    </div>
  </div>;
};

const ThingieWithTitle = ({ title, ...others }) => {
  <Thingie {...others}>
    <Title value={title} />
  </Thingie>;
};

const ThingieWithTitleAndText = ({ title, ...others }) => {
  <Thingie {...others}>
    <Title value={title} />
    <Text />
  </Thingie>;
};
```

## Be Careful with setting state

Always set state as a function of the previous state if the new state relies on the previous state. React state updates can be batched, and not writing your updates this way can lead to unexpected results.

```javascript
// bad
import React, { useState } from 'react';

export const PreviousStateBad = () => {
  const [isDisabled, setIsDisabled] = useState(false);

  const toggleButton = () => setIsDisabled(!isDisabled);

  const toggleButton2Times = () => {
    for (let i = 0; i < 2; i++) {
      toggleButton();
    }
  };

  return (
    <div>
      <button disabled={isDisabled}>
        I'm {isDisabled ? 'disabled' : 'enabled'}
      </button>
      <button onClick={toggleButton}>Toggle button state</button>
      <button onClick={toggleButton2Times}>Toggle button state 2 times</button>
    </div>
  );
};

// goood

import React, { useState } from 'react';

export const PreviousStateGood = () => {
  const [isDisabled, setIsDisabled] = useState(false);

  const toggleButton = () => setIsDisabled((isDisabled) => !isDisabled);

  const toggleButton2Times = () => {
    for (let i = 0; i < 2; i++) {
      toggleButton();
    }
  };

  return (
    <div>
      <button disabled={isDisabled}>
        I'm {isDisabled ? 'disabled' : 'enabled'}
      </button>
      <button onClick={toggleButton}>Toggle button state</button>
      <button onClick={toggleButton2Times}>Toggle button state 2 times</button>
    </div>
  );
};
```

## Keep JSX Clean from JS

```javascript
import useFetchPosts from '../hooks/useFetchPosts.js';

export default function FeaturedPosts() {
  const posts = useFetchPosts()

  return (
    <ul>
      {posts.map((post) => (
        <li onClick={event => {
          console.log(event.target, 'clicked!');
        }} key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}



// src/components/FeaturedPosts.js

import useFetchPosts from '../hooks/useFetchPosts.js';

export default function FeaturedPosts() {
  const posts = useFetchPosts()

  function handlePostClick(event) {
    console.log(event.target, 'clicked!');
  }

  return (
    <ul>
      {posts.map((post) => (
        <li onClick={handlePostClick} key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

## Use Conditional Rendering

```javascript
import React, { useState } from 'react'

// one condition
export const ConditionalRenderingWhenTrueGood = () => {
  const [showConditionalText, setShowConditionalText] = useState(false)

  const handleClick = () =>
    setShowConditionalText(showConditionalText => !showConditionalText)

  return (
    <div>
      <button onClick={handleClick}>Toggle the text</button>
      {showConditionalText && <p>The condition must be true!</p>}
    </div>
  )
}


import React, { useState } from 'react'

export const ConditionalRenderingGood = () => {
  const [showConditionOneText, setShowConditionOneText] = useState(false)

  const handleClick = () =>
    setShowConditionOneText(showConditionOneText => !showConditionOneText)

  return (
    <div>
      <button onClick={handleClick}>Toggle the text</button>
      {showConditionOneText ? (
        <p>The condition must be true!</p>
      ) : (
        <p>The condition must be false!</p>
      )}
    </div>
  )
```

## Use Query Library

Use Query Library like React query or SWR, they help us track loading state, handle errors, re-fetch stuff, give ways on interval, caching, mutations

useEffect approach. Here we are waiting for component to be mounted to start fetching data, which can take some time. Also, we creating unneeded states for loading and erorr, and hanlding it on our own.

```javascript
import { useEffect, useState } from 'react';

export const WithUseEffect = () => {
  const [products, setProducts] = useState([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const fetchProducts = async () => {
    try {
      setIsLoading(true);
      const response = await fetch('https://fakestoreapi.com/products');
      const products = await response.json();
      console.log('products', products);
      if (products) {
        setProducts(products);
      }
    } catch (e) {
      const error = e as Error;
      setError(error);
    }
    setIsLoading(false);
  };

  useEffect(() => {
    console.log('Component mounted.');
    fetchProducts();
  }, []);

  if (isLoading) {
    return <p>Loading...</p>;
  }

  if (error) {
    return <p>{error.message}</p>;
  }

  return (
    <div>
      {products.length > 0 && (
        <ul>
          {products.map((product: any) => (
            <li key={product.id}>{product.title}</li>
          ))}
        </ul>
      )}
    </div>
  );
};
```

useQuery approach. As you see code is much more simpler and readable. React Query takes care of errors and loading states, and also it start to fetch the data when the component is starting to render, so we have a bit more time to fetch it.

```javascript
import { useQuery } from '@tanstack/react-query';

export const WithReactQuery = () => {
  const fetchProducts = async () => {
    const response = await fetch('https://fakestoreapi.com/products');
    const products = await response.json();
    console.log('products', products);
    return products;
  };

  const {
    isError,
    isLoading,
    data: products,
    error,
  } = useQuery(['products'], fetchProducts, { staleTime: 6000 });

  if (isLoading) {
    return <p>Loading...</p>;
  }

  return (
    <div>
      {products && (
        <ul>
          {products.map((product: any) => (
            <li key={product.id}>{product.title}</li>
          ))}
        </ul>
      )}
    </div>
  );
};
```

## Don’t use Redux unless you need it

Couple years back, when React introduced hooks and new reactive state management model,
the idea of requiring efectively an external state manager like Redux, didnt become as important. You can now go and use Context and Hooks to maintain state globaly and closer to where you actually use it.

Better to start with React Hooks, and see how far they can get you.
If it is not enough, you can use Query Library, like React Query to cache and manage state
For forms you can use react hook forms or formik and if this combination is not enough, consider Redux, especiialy Redux Toolkit. But also take in a considearration atomic state managers like Jotai, or more simplified Zustand You dont need to go for redux out of the box. keep it simple, use as much as you need, but not anymore.

## Dont build your own UI library

React has an amazing set of awesome UI libraries, there is MUI, Bootstrap, Chakra and more
libraries, that out of the box come with most of controls you will ever need to put on a page.
Plus they are accesible, themeable, scalable, got great docs and examples and community
around them.
