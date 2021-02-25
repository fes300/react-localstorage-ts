![release](https://github.com/ModataSRL/react-localstorage-ts/actions/workflows/release.yml/badge.svg)

# react-localstorage-ts

A small layer over the browser's localstorage, fallbacks to an in-memory store localstorage is not supported by the browser.  

Built on `io-ts` and `fp-ts`, `react-localstorage-ts` gives you a standard way to access objects stored locally using `io-ts`'s encoding/decoding abilities.

## install

### yarn
```shell
yarn add react-localstorage-ts fp-ts io-ts monocle-ts newtype-ts
```
### npm
```shell
npm install react-localstorage-ts fp-ts io-ts monocle-ts newtype-ts
```

## quick start
To use `react-localstorage-ts` you have to follow a few simple steps:

First you need to define the list of "storable" items by expanding the `StoredItems` interface:

```ts
// store.ts
import * as t from "io-ts"
export const ThemeFlavour = t.union([t.literal("light"), t.literal("dark")])
export type ThemeFlavour = t.TypeOf<typeof ThemeFlavour>

declare module "react-localstorage-ts" {
    interface StoredItems {
        access_token: string
        theme: ThemeFlavour
    }
}
```

then you create the hooks to read/write the values you just defined:

```tsx
// localHooks.ts
import * as t from "io-ts"
import {
  makeDefaultedUseLocalItem,
  makeUseLocalItem,
} from "react-localstorage-ts"
import {ThemeFlavour} from "./store"

export const useThemeFlavour = makeDefaultedUseLocalItem("theme", ThemeFlavour, () => "light")
export const useAccessToken = makeUseLocalItem("access_token", t.string)
```

then you use them in your react components:
```tsx
// App.tsx
import * as React from "react"
import * as E from "fp-ts/Either"
import LightThemeApp from "./components/LightThemeApp"
import DarkThemeApp from "./components/DarkThemeApp"
import { useThemeFlavour } from "./localHooks"

const App: React.FC = () => {
  const [theme, setTheme] = useThemeFlavour()
  
  return pipe(
    theme,
    E.fold(
      () => {
        console.error('wrong value stored in localStorage!')
      },
      themeFlavour => {
        switch (themeFlavour) {
          case "light": {
            return <LightThemeApp />
          }
          case "dark": {
            return <DarkThemeApp />
          }
        }
      }
    )
  )
}

export default App
```
## LocalValue
A new data structure is defined for items stored in localstorage, `LocalValue`. When dealing with a value stored in your localstorage there are three possibilities:
1. there is no value in your localstorage (optionality).
2. the value is present, but it is wrong (correctness).
3. the value is present and it is valid (also correctness).

The structure of LocalValue represent the optionality/correctness dicotomy by using well known contructs, `Option` and `Either`:

```ts
import * as t from "io-ts"

type LocalValue<V> = O.Option<E.Either<t.Errors, V>>
```
It also has instances for some of the most common type-classes
and you can use it in the same way you usually use your usual `fp-ts` abstractions:

```tsx
// LoginLayout.tsx
import * as LV from "react-localstorage-ts/LocalValue"
import { useAccessToken } from "./localHooks"
import { goToLoginPage } from "./router"
import App from "./App"

const LoginLayout: React.FC = ({ children }) => {
  const [token] = useAccessToken()

  React.useEffect(() => {
    if (!LV.isValid) {
      goToLoginPage()
    }
  }, [])
  
  return pipe(
    token,
    LV.fold(() => null, () => null, () => <>{ children }</>),
  )
}

// LoginPage.tsx
import * as LV from "react-localstorage-ts/LocalValue"
import { goToHomePage } from "./router"
import { useAccessToken } from "./localHooks"

const LoginPage: React.FC = ({ children }) => {
  const [token, setToken] = useAccessToken()

  React.useEffect(() => {
    if (LV.isValid) {
      goToHomePage()
    }
  }, [])

  return (
    <Form
      onSubmit={() => api.getToken.then(t => setToken(t))}
    />
  )
}
```

**N.B.** when you use `makeDefaultedUseLocalItem`, you loose the optionality of your value, so you are left with an `Either` instead of a `LocalValue`. 

## defining codecs 
creating a custom codec to use with `makeUseLocalItem` can be a really non-trivial task, that's why 
we ship the utility codec `JSONFromString` that can be used to "ease the pain". Here is an example of how you can use it to define a custom codec:

```ts
import * as t from "io-ts"
import * as E from "fp-ts/Either"
import { pipe } from "fp-ts/function"
import { JSONFromString, isoJSON } from "react-localstorage-ts/JSONFromString"

const ShapeCodec = t.type({ s: t.string, d: DateFromISOString })
type ShapeCodec = t.TypeOf<typeof ShapeCodec>

const defaultShape: ShapeCodec = {
  s: "foo",
  d: new Date(),
}

export const ShapeCodecFromString = new t.Type<ShapeCodec, string>(
  "ShapeCodecFromString",
  ShapeCodec.is,
  (u, c) => {
    return pipe(
      JSONFromString.validate(u, c),
      E.chain((json) => ShapeCodec.validate(json, c)),
    )
  },
  (v) => {
    return pipe(v, ShapeCodec.encode, isoJSON.wrap, JSONFromString.encode)
  },
)
```

## contributing
to commit to this repository there are a few rules:
- your commits must follow the conventional commit standard (it should be enforced by husky `commit-msg` hook).
- your code must be formatted using prettier. 
- all tests must pass.

## release flow
[here](https://github.com/semantic-release/semantic-release/blob/1405b94296059c0c6878fb8b626e2c5da9317632/docs/recipes/pre-releases.md) you can find an explanation of the release flow.